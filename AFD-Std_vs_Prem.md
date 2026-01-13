# Azure Front Door Standard vs Premium — Detailed Comparison

---

## 1. Core Platform (Both SKUs)

| Feature | Standard | Premium |
|--------|----------|----------|
| Global Anycast edge network | ✅ | ✅ |
| Layer 7 load balancing | ✅ | ✅ |
| HTTP/HTTPS endpoints | ✅ | ✅ |
| Custom domains + TLS | ✅ | ✅ |
| Path-based routing | ✅ | ✅ |
| Rules Engine (rewrite, redirect, headers, etc.) | ✅ | ✅ |
| Caching | ✅ | ✅ |
| Compression | ✅ | ✅ |
| Health probes | ✅ | ✅ |
| Multi-origin / failover | ✅ | ✅ |

---

## 2. Rules Engine (Traffic Behavior)

| Feature | Standard | Premium |
|--------|----------|----------|
| Create rule sets | ✅ | ✅ |
| Redirect / Rewrite | ✅ | ✅ |
| Add/Remove headers | ✅ | ✅ |
| Conditional routing | ✅ | ✅ |
| Cache control rules | ✅ | ✅ |

---

## 3. WAF (Web Application Firewall) — BIG DIFFERENCE

### 3.1 Managed Protection

| Feature | Standard | Premium |
|--------|----------|----------|
| Microsoft DefaultRuleSet | ❌ | ✅ |
| OWASP rule sets | ❌ | ✅ |
| Bot protection | ❌ | ✅ |
| Microsoft threat intelligence | ❌ | ✅ |
| Auto-updated signatures | ❌ | ✅ |

### 3.2 Custom WAF Rules

| Feature | Standard | Premium |
|--------|----------|----------|
| IP allow/deny | ✅ | ✅ |
| Geo filtering | ✅ | ✅ |
| Header / cookie / path match | ✅ | ✅ |
| Rate limiting | ✅ | ✅ |
| Custom block/allow rules | ✅ | ✅ |

---

## 4. Private Link to Origins (Critical for ZTNA / Private Apps)

| Feature | Standard | Premium |
|--------|----------|----------|
| Public internet origins | ✅ | ✅ |
| Private Link origins (private backend access) | ❌ | ✅ |

---

## 5. Zero Trust / Enterprise Security

| Feature | Standard | Premium |
|--------|----------|----------|
| mTLS to origin | ❌ | ✅ |
| Advanced security integration | ❌ | ✅ |
| Locked-down private origins | ❌ | ✅ |

---

## 6. Logging, Metrics, Diagnostics

| Feature | Standard | Premium |
|--------|----------|----------|
| Access logs | ✅ | ✅ |
| WAF logs | ✅ | ✅ |
| Azure Monitor integration | ✅ | ✅ |

---

## 7. Pricing Model

| Item | Standard | Premium |
|------|----------|----------|
| Base monthly fee | ✅ ~ $35/month | ✅ ~ $330/month |
| Pay per request | ✅ | ✅ |
| Pay per GB transfer | ✅ | ✅ |
| WAF processing | Usage-based | Usage-based |

---

## 8. When You MUST Use Premium

| Requirement | Use |
|------------|------|
| DefaultRuleSet / OWASP | Premium |
| Bot protection | Premium |
| Enterprise WAF | Premium |
| Private Link origins | Premium |
| ZTNA-style architecture | Premium |
| Private PaaS / private APIs behind Front Door | Premium |


## Official document
- https://learn.microsoft.com/en-us/azure/frontdoor/understanding-pricing
- https://learn.microsoft.com/en-us/azure/frontdoor/billing
