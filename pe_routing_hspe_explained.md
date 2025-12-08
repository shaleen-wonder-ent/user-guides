# Azure Private Endpoint /32 Routing and HSPE Deep Dive

## What is a /32 Route in a VNet?

A **/32 route** represents the most specific IPv4 route possible—it targets exactly one IP address.  
Azure automatically creates a `/32` **system route** for every Private Endpoint (PE), ensuring traffic is sent directly to the PE interface.

Example:
```
10.5.4.12/32 → InterfaceEndpoint (system route)
```

This overrides any broader routes like `/24`, `/16`, or `0.0.0.0/0` because Azure uses the **longest prefix match**.

---

## Behavior Before and After RouteTables Enabled

### Before enabling RouteTables:
- Private Endpoints **ignore** User-Defined Routes (UDRs).
- Traffic always goes directly to the PE through Azure's `/32` system route.
- Firewalls/NVAs cannot inspect or intercept PE traffic.

### After enabling RouteTables:
- Private Endpoints start **honoring UDRs**.
- System `/32` route still exists and still wins — unless overridden by a matching `/32` UDR.
- This allows advanced routing scenarios (e.g., forcing PE traffic through a firewall).

---

## Overriding Azure’s /32 Route

Azure allows you to override the system `/32` route by creating your own **/32 UDR**:

```
10.5.4.12/32 → Firewall
```

Routing precedence rule:
> **If prefix length is equal, UDR overrides system routes.**

This forces traffic:
VM/App → Firewall → Private Endpoint

---

## Visual Diagrams

### 1. Default Behavior (Before RouteTables Enabled)
Private Endpoints ignore UDRs.

```
+-------------------+         Azure Backbone          +----------------------+
|   VM / App        |-------------------------------->| Private Endpoint     |
|   10.0.1.10       |   (PE ignores UDRs)             | 10.5.4.12 (/32)      |
+-------------------+                                 +----------------------+
                          Direct /32 system route
```

---

### 2. After RouteTables Enabled (Normal Behavior)
PE honors UDRs but system `/32` still wins unless overridden.

```
+-------------------+         Azure Backbone          +----------------------+
|   VM / App        |-------------------------------->| Private Endpoint     |
|   10.0.1.10       |  (UDRs honored, but /32 wins)   | 10.5.4.12 (/32)      |
+-------------------+                                 +----------------------+

Route Table:
  10.5.4.12/32 → InterfaceEndpoint   (System Route — most specific)
  0.0.0.0/0    → Firewall            (UDR)
```

---

### 3. After Adding Your Own /32 UDR (Forcing Firewall Path)
Your /32 UDR overrides Azure’s system /32 route.

```
+-------------------+        +-----------------+        +----------------------+
|   VM / App        |------->|   Firewall/NVA  |------->| Private Endpoint     |
|   10.0.1.10       |  /32   | 10.0.100.4      | /32    | 10.5.4.12 (/32)      |
+-------------------+        +-----------------+        +----------------------+

Route Table (with override):
  10.5.4.12/32 → Firewall         (UDR — overrides system)
  10.5.4.12/32 → InterfaceEndpoint (System)
```

---

## Summary

- Azure always injects a `/32` system route for Private Endpoints.
- When RouteTables are enabled, Private Endpoints begin evaluating UDRs.
- System `/32` route normally wins due to specificity.
- Creating your own `/32` UDR lets you force PE traffic through a firewall.
- This scenario is advanced and requires firewall configuration to correctly forward Private Link traffic.

---

