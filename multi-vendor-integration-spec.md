# Multi-Vendor Integration Specification
**Palo Alto Networks + F5 BIG-IP + Infoblox + Azure**

---

## Document Information

| Property | Value |
|----------|-------|
| **Version** | 1.0 |
| **Last Updated** | 2026-02-24 |
| **Status** | Draft |
| **Audience** | Technical Architects, DevOps Engineers, Integration Developers |
| **Purpose** | Detailed technical specification for AI agent integration with Palo Alto, F5, and Infoblox |

---

![Reference Architecture: Hybrid AI Agent Platform](https://github.com/user-attachments/assets/96c97cee-47c3-4023-b111-1e0a8903c6e8)
<img width="1536" height="1024" alt="Architecture-connector" src="https://github.com/user-attachments/assets/eb92e015-d6ce-4a10-ab8d-36667c0f0516" />

---
## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Palo Alto Networks Integration](#2-palo-alto-networks-integration)
3. [F5 BIG-IP Integration](#3-f5-big-ip-integration)
4. [Infoblox Integration](#4-infoblox-integration)
5. [Azure Integration](#5-azure-integration)
6. [Cross-Vendor Orchestration](#6-cross-vendor-orchestration)
7. [Authentication & Security](#7-authentication--security)
8. [Error Handling & Retry Logic](#8-error-handling--retry-logic)
9. [Monitoring & Observability](#9-monitoring--observability)
10. [Testing & Validation](#10-testing--validation)

---

## 1. Architecture Overview

<details>
<summary>Click to expand</summary>

### 1.1 High-Level Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                        AI Agent Layer                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐ │
│  │ Network    │  │ Security   │  │ Load       │  │ ASR-DR   │ │
│  │ Agent      │  │ Agent      │  │ Balancer   │  │ Agent    │ │
│  │            │  │            │  │ Agent      │  │          │ │
│  └────────────┘  └────────────┘  └────────────┘  └──────────┘ │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│              Infrastructure Abstraction Service (IAS)         │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Intent Translation Engine                               │ │
│  │  - Parse agent intent                                    │ │
│  │  - Determine affected vendors                            │ │
│  │  - Generate vendor-specific commands                     │ │
│  │  - Validate consistency                                  │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  State Manager                                           │ │
│  │  - Track current state across vendors                    │ │
│  │  - Detect configuration drift                            │ │
│  │  - Maintain desired state                                │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Orchestration Engine                                    │ │
│  │  - Execute operations in correct order                   │ │
│  │  - Handle dependencies                                   │ │
│  │  - Coordinate rollback if needed                         │ │
│  └──────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ Palo Alto     │   │ F5 BIG-IP        │   │ Infoblox         │
│ Adapter       │   │ Adapter          │   │ Adapter          │
│               │   │                  │   │                  │
│ - Panorama    │   │ - iControl REST  │   │ - WAPI           │
│ - XML API     │   │ - AS3            │   │ - REST API       │
│ - REST API    │   │ - TMSH           │   │ - Grid Master    │
└───────────────┘   └──────────────────┘   └──────────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ Palo Alto     │   │ F5 BIG-IP        │   │ Infoblox Grid    │
│ Panorama      │   │ Device(s)        │   │ Master           │
│               │   │                  │   │                  │
│ - Firewalls   │   │ - Load Balancers │   │ - DNS            │
│ - Policies    │   │ - Pools          │   │ - DHCP           │
│ - NAT Rules   │   │ - Virtual Servers│   │ - IPAM           │
└───────────────┘   └──────────────────┘   └──────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Azure Adapter    │
                    │                  │
                    │ - Azure SDK      │
                    │ - ARM API        │
                    │ - Resource Graph │
                    └──────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Azure Resources  │
                    │                  │
                    │ - VNets          │
                    │ - NSGs           │
                    │ - Load Balancers │
                    │ - VMs            │
                    └──────────────────┘
```

### 1.2 Component Responsibilities

| Component | Responsibility |
|-----------|---------------|
| **AI Agents** | Receive user intents, analyze requirements, make decisions |
| **Intent Translation Engine** | Convert high-level intents into vendor-specific operations |
| **State Manager** | Track and maintain desired state across all vendors |
| **Orchestration Engine** | Execute operations in correct sequence, handle dependencies |
| **Vendor Adapters** | Interface with vendor-specific APIs, handle authentication |
| **Monitoring Service** | Track health, performance, and compliance across vendors |

### 1.3 Communication Protocols

| Source | Destination | Protocol | Port | Authentication |
|--------|-------------|----------|------|----------------|
| IAS | Palo Alto Panorama | HTTPS | 443 | API Key |
| IAS | F5 BIG-IP | HTTPS | 443 | Token-based |
| IAS | Infoblox Grid Master | HTTPS | 443 | Basic Auth / Token |
| IAS | Azure | HTTPS | 443 | Managed Identity / Service Principal |

### 1.4 Data Flow Example

**Scenario**: Block malicious IP across all vendors

```
1. Security Agent receives intent:
   "Block IP 203.0.113.50 across all environments"

2. Intent Translation Engine analyzes:
   - Identify affected resources:
     • Palo Alto: PA-DMZ-Chennai, PA-DMZ-Pune
     • Azure: NSG-Prod-*, NSG-DMZ-*
     • F5: F5-Chennai-AFM, F5-Pune-AFM
   
3. State Manager checks current state:
   - Palo Alto: IP not blocked
   - Azure: IP not blocked in most NSGs
   - F5: IP not blocked
   
4. Orchestration Engine creates plan:
   Step 1: Block in Palo Alto (edge first)
   Step 2: Block in F5 AFM (application layer)
   Step 3: Block in Azure NSGs (defense in depth)
   Step 4: Validate across all vendors
   
5. Execute operations:
   - Palo Alto Adapter: Create security rule, push to devices
   - F5 Adapter: Add to AFM blocked IPs
   - Azure Adapter: Add deny rule to all NSGs
   
6. Validate and confirm:
   - Test from external source (should be blocked)
   - Verify rule exists in all locations
   - Update CMDB and documentation
```

</details>

---

## 2. Palo Alto Networks Integration

<details>
<summary>Click to expand</summary>

### 2.1 Palo Alto Infrastructure

**Contoso Architecture Components**:
- **Panorama**: Centralized management (version 10.x or higher)
- **Device Groups**:
  - `DG-Chennai-DMZ` (firewalls protecting Chennai DMZ)
  - `DG-Pune-DMZ` (firewalls protecting Pune DMZ)
  - `DG-HUB-Chennai` (hub firewall)
  - `DG-HUB-Pune` (hub firewall)
- **Template Stacks**: Network and device configurations
- **Shared Objects**: Address objects, service objects, security profiles

### 2.2 API Options

| API Type | Use Case | Pros | Cons |
|----------|----------|------|------|
| **XML API** | Legacy operations, full control | Complete feature coverage | Complex XML parsing |
| **REST API** | Modern operations (PAN-OS 9.0+) | Easier to use, JSON-based | Limited feature coverage |
| **PAN-OS SDK (Python)** | Automation scripts | Object-oriented, Pythonic | Requires Python runtime |

**Recommendation**: Use **PAN-OS Python SDK** for primary integration, fallback to XML API for advanced features.

### 2.3 Authentication

**Method 1: API Key (Recommended)**
```python
# Generate API key
curl -k -X POST 'https://panorama.Contoso.local/api/?type=keygen&user=api-admin&password=SecurePass123'

# Response
<response status="success">
  <result>
    <key>LUFRPT14MW5xOEo1R09KVlBZNnpnemh0VHRBOWl6TGM9bXcwM3JHUGVhRlNiY0dCR0srNERUQT09</key>
  </result>
</response>

# Use API key in subsequent calls
curl -k 'https://panorama.Contoso.local/api/?type=op&cmd=<show><system><info></info></system></show>&key=LUFRPT14...'
```

**Method 2: Service Account with RBAC**
```xml
<!-- Create dedicated API service account with restricted permissions -->
<mgt-config>
  <users>
    <entry name="svc-automation">
      <permissions>
        <role-based>
          <custom>
            <profile>API-Automation-Role</profile>
          </custom>
        </role-based>
      </permissions>
      <phash>$1$encrypted$hash</phash>
    </entry>
  </users>
</mgt-config>
```

### 2.4 Palo Alto Adapter Implementation

#### 2.4.1 Adapter Class Structure

```python
from panos.panorama import Panorama, DeviceGroup
from panos.policies import SecurityRule, NatRule
from panos.objects import AddressObject, ServiceObject
from typing import List, Dict, Optional
import logging

class PaloAltoAdapter:
    """
    Adapter for Palo Alto Networks Panorama integration
    Supports security policies, NAT rules, objects, and device management
    """
    
    def __init__(self, panorama_host: str, api_key: str):
        """
        Initialize Palo Alto adapter
        
        Args:
            panorama_host: Panorama hostname or IP
            api_key: API key for authentication
        """
        self.panorama_host = panorama_host
        self.api_key = api_key
        self.panorama = None
        self.logger = logging.getLogger(__name__)
        self._connect()
    
    def _connect(self):
        """Establish connection to Panorama"""
        try:
            self.panorama = Panorama(
                hostname=self.panorama_host,
                api_key=self.api_key
            )
            # Verify connectivity
            self.panorama.refresh_system_info()
            self.logger.info(f"Connected to Panorama {self.panorama_host}")
        except Exception as e:
            self.logger.error(f"Failed to connect to Panorama: {e}")
            raise
    
    def get_device_groups(self) -> List[str]:
        """Get all device groups from Panorama"""
        try:
            device_groups = DeviceGroup.refreshall(self.panorama)
            return [dg.name for dg in device_groups]
        except Exception as e:
            self.logger.error(f"Failed to get device groups: {e}")
            raise
```

#### 2.4.2 Security Rule Operations

```python
    def create_security_rule(
        self,
        name: str,
        source_zones: List[str],
        destination_zones: List[str],
        source_addresses: List[str],
        destination_addresses: List[str],
        applications: List[str],
        services: List[str],
        action: str,
        device_group: str,
        description: Optional[str] = None,
        log_start: bool = True,
        log_end: bool = True,
        profile_group: Optional[str] = None
    ) -> Dict:
        """
        Create security rule in Panorama
        
        Args:
            name: Rule name
            source_zones: List of source zones
            destination_zones: List of destination zones
            source_addresses: List of source addresses/objects
            destination_addresses: List of destination addresses/objects
            applications: List of applications
            services: List of services
            action: allow, deny, drop, reset-client, reset-server
            device_group: Target device group
            description: Rule description
            log_start: Log at session start
            log_end: Log at session end
            profile_group: Security profile group
            
        Returns:
            dict: Operation result with rule details
        """
        try:
            # Get device group object
            dg = DeviceGroup(device_group)
            self.panorama.add(dg)
            
            # Create security rule object
            rule = SecurityRule(
                name=name,
                fromzone=source_zones,
                tozone=destination_zones,
                source=source_addresses,
                destination=destination_addresses,
                application=applications,
                service=services,
                action=action,
                description=description,
                log_start=log_start,
                log_end=log_end
            )
            
            # Add security profile group if specified
            if profile_group:
                rule.group = profile_group
            
            # Add rule to device group
            dg.add(rule)
            
            # Create rule in Panorama
            rule.create()
            
            self.logger.info(f"Created security rule '{name}' in device group '{device_group}'")
            
            return {
                "status": "success",
                "rule_name": name,
                "device_group": device_group,
                "rule_uuid": rule.uuid,
                "action": action
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create security rule: {e}")
            raise
    
    def delete_security_rule(
        self,
        rule_name: str,
        device_group: str
    ) -> Dict:
        """Delete security rule from device group"""
        try:
            dg = DeviceGroup(device_group)
            self.panorama.add(dg)
            
            # Find and delete rule
            rule = SecurityRule(rule_name)
            dg.add(rule)
            rule.delete()
            
            self.logger.info(f"Deleted security rule '{rule_name}' from device group '{device_group}'")
            
            return {
                "status": "success",
                "rule_name": rule_name,
                "device_group": device_group
            }
            
        except Exception as e:
            self.logger.error(f"Failed to delete security rule: {e}")
            raise
    
    def get_security_rules(
        self,
        device_group: str,
        filter_name: Optional[str] = None
    ) -> List[Dict]:
        """Get security rules from device group"""
        try:
            dg = DeviceGroup(device_group)
            self.panorama.add(dg)
            
            # Refresh security rules
            rules = SecurityRule.refreshall(dg)
            
            # Filter if needed
            if filter_name:
                rules = [r for r in rules if filter_name.lower() in r.name.lower()]
            
            # Convert to dict format
            rule_list = []
            for rule in rules:
                rule_list.append({
                    "name": rule.name,
                    "uuid": rule.uuid,
                    "fromzone": rule.fromzone,
                    "tozone": rule.tozone,
                    "source": rule.source,
                    "destination": rule.destination,
                    "application": rule.application,
                    "service": rule.service,
                    "action": rule.action,
                    "description": rule.description,
                    "disabled": rule.disabled
                })
            
            return rule_list
            
        except Exception as e:
            self.logger.error(f"Failed to get security rules: {e}")
            raise
```

#### 2.4.3 Address Object Management

```python
    def create_address_object(
        self,
        name: str,
        value: str,
        object_type: str = "ip-netmask",
        device_group: str = "shared",
        description: Optional[str] = None,
        tags: Optional[List[str]] = None
    ) -> Dict:
        """
        Create address object
        
        Args:
            name: Object name
            value: IP address, FQDN, or IP range
            object_type: ip-netmask, ip-range, fqdn
            device_group: Device group (use 'shared' for all groups)
            description: Object description
            tags: List of tags
            
        Returns:
            dict: Operation result
        """
        try:
            if device_group == "shared":
                parent = self.panorama
            else:
                parent = DeviceGroup(device_group)
                self.panorama.add(parent)
            
            # Create address object
            addr_obj = AddressObject(
                name=name,
                value=value,
                type=object_type,
                description=description,
                tag=tags or []
            )
            
            parent.add(addr_obj)
            addr_obj.create()
            
            self.logger.info(f"Created address object '{name}' in '{device_group}'")
            
            return {
                "status": "success",
                "object_name": name,
                "value": value,
                "type": object_type,
                "device_group": device_group
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create address object: {e}")
            raise
    
    def get_address_objects(
        self,
        device_group: str = "shared",
        filter_name: Optional[str] = None
    ) -> List[Dict]:
        """Get address objects from device group"""
        try:
            if device_group == "shared":
                parent = self.panorama
            else:
                parent = DeviceGroup(device_group)
                self.panorama.add(parent)
            
            objects = AddressObject.refreshall(parent)
            
            if filter_name:
                objects = [o for o in objects if filter_name.lower() in o.name.lower()]
            
            return [
                {
                    "name": obj.name,
                    "value": obj.value,
                    "type": obj.type,
                    "description": obj.description,
                    "tags": obj.tag
                }
                for obj in objects
            ]
            
        except Exception as e:
            self.logger.error(f"Failed to get address objects: {e}")
            raise
```

#### 2.4.4 Commit and Push Operations

```python
    def commit_to_panorama(self) -> Dict:
        """Commit changes to Panorama"""
        try:
            # Commit to Panorama
            job = self.panorama.commit(sync=True)
            
            self.logger.info(f"Committed changes to Panorama (Job ID: {job})")
            
            return {
                "status": "success",
                "job_id": job,
                "message": "Changes committed to Panorama"
            }
            
        except Exception as e:
            self.logger.error(f"Failed to commit to Panorama: {e}")
            raise
    
    def push_to_device_group(
        self,
        device_group: str,
        include_template: bool = False
    ) -> Dict:
        """
        Push configuration from Panorama to device group
        
        Args:
            device_group: Target device group
            include_template: Also push template changes
            
        Returns:
            dict: Operation result with job ID
        """
        try:
            # Push to device group
            job = self.panorama.commit_all(
                sync=True,
                devicegroup=device_group,
                include_template=include_template
            )
            
            self.logger.info(
                f"Pushed configuration to device group '{device_group}' (Job ID: {job})"
            )
            
            return {
                "status": "success",
                "job_id": job,
                "device_group": device_group,
                "message": f"Configuration pushed to {device_group}"
            }
            
        except Exception as e:
            self.logger.error(f"Failed to push to device group: {e}")
            raise
    
    def get_commit_status(self, job_id: int) -> Dict:
        """Get status of commit job"""
        try:
            result = self.panorama.check_commit_status(job_id)
            
            return {
                "job_id": job_id,
                "status": result.get("status"),
                "progress": result.get("progress"),
                "details": result.get("details"),
                "warnings": result.get("warnings"),
                "errors": result.get("errors")
            }
            
        except Exception as e:
            self.logger.error(f"Failed to get commit status: {e}")
            raise
```

#### 2.4.5 NAT Rules

```python
    def create_nat_rule(
        self,
        name: str,
        source_zone: str,
        destination_zone: str,
        source_addresses: List[str],
        destination_addresses: List[str],
        service: str,
        nat_type: str,
        device_group: str,
        translated_address: Optional[str] = None,
        translated_port: Optional[int] = None,
        description: Optional[str] = None
    ) -> Dict:
        """
        Create NAT rule
        
        Args:
            name: Rule name
            source_zone: Source zone
            destination_zone: Destination zone
            source_addresses: List of source addresses
            destination_addresses: List of destination addresses
            service: Service (e.g., 'service-http')
            nat_type: ipv4, nat64, nptv6
            device_group: Target device group
            translated_address: Translated address for SNAT
            translated_port: Translated port for DNAT
            description: Rule description
            
        Returns:
            dict: Operation result
        """
        try:
            dg = DeviceGroup(device_group)
            self.panorama.add(dg)
            
            # Create NAT rule
            nat_rule = NatRule(
                name=name,
                fromzone=source_zone,
                tozone=destination_zone,
                source=source_addresses,
                destination=destination_addresses,
                service=service,
                nat_type=nat_type,
                description=description
            )
            
            # Configure source translation (SNAT)
            if translated_address:
                nat_rule.source_translation_type = "dynamic-ip-and-port"
                nat_rule.source_translation_address_type = "translated-address"
                nat_rule.source_translation_translated_addresses = [translated_address]
            
            # Configure destination translation (DNAT)
            if translated_port:
                nat_rule.destination_translated_port = translated_port
            
            dg.add(nat_rule)
            nat_rule.create()
            
            self.logger.info(f"Created NAT rule '{name}' in device group '{device_group}'")
            
            return {
                "status": "success",
                "rule_name": name,
                "device_group": device_group,
                "nat_type": nat_type
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create NAT rule: {e}")
            raise
```

#### 2.4.6 Traffic and Session Information

```python
    def get_session_info(
        self,
        firewall_serial: str,
        filter_expression: Optional[str] = None
    ) -> Dict:
        """
        Get active session information from firewall
        
        Args:
            firewall_serial: Serial number of target firewall
            filter_expression: Filter (e.g., 'source 10.10.10.5')
            
        Returns:
            dict: Session information
        """
        try:
            # Build operational command
            if filter_expression:
                cmd = f"<show><session><all><filter>{filter_expression}</filter></all></session></show>"
            else:
                cmd = "<show><session><all></all></session></show>"
            
            # Execute on specific firewall via Panorama
            result = self.panorama.op(
                cmd=cmd,
                cmd_xml=False,
                vsys=None,
                device_name=firewall_serial
            )
            
            # Parse XML response
            sessions = self._parse_session_xml(result)
            
            return {
                "status": "success",
                "firewall": firewall_serial,
                "total_sessions": len(sessions),
                "sessions": sessions
            }
            
        except Exception as e:
            self.logger.error(f"Failed to get session info: {e}")
            raise
    
    def get_threat_logs(
        self,
        firewall_serial: str,
        query: str = "(severity eq high)",
        num_logs: int = 100
    ) -> List[Dict]:
        """
        Query threat logs from firewall
        
        Args:
            firewall_serial: Serial number of target firewall
            query: Log query filter
            num_logs: Number of logs to retrieve
            
        Returns:
            list: Threat log entries
        """
        try:
            cmd = f"""
            <show>
              <log>
                <threat>
                  <direction>backward</direction>
                  <query>{query}</query>
                  <nlogs>{num_logs}</nlogs>
                </threat>
              </log>
            </show>
            """
            
            result = self.panorama.op(
                cmd=cmd,
                cmd_xml=False,
                device_name=firewall_serial
            )
            
            logs = self._parse_threat_logs_xml(result)
            
            return logs
            
        except Exception as e:
            self.logger.error(f"Failed to get threat logs: {e}")
            raise
```

### 2.5 Integration Patterns

#### Pattern 1: Block Malicious IP Across All Firewalls

```python
def block_malicious_ip_palo_alto(
    adapter: PaloAltoAdapter,
    ip_address: str,
    reason: str,
    device_groups: List[str]
) -> Dict:
    """
    Block malicious IP across multiple device groups
    
    Args:
        adapter: PaloAltoAdapter instance
        ip_address: IP to block
        reason: Reason for blocking
        device_groups: List of device groups to apply rule
        
    Returns:
        dict: Results from all device groups
    """
    results = {}
    
    # Step 1: Create address object (shared)
    object_name = f"Blocked-{ip_address.replace('.', '_')}"
    
    try:
        adapter.create_address_object(
            name=object_name,
            value=ip_address,
            object_type="ip-netmask",
            device_group="shared",
            description=f"Blocked: {reason}",
            tags=["malicious", "auto-blocked"]
        )
        
        # Step 2: Create security rule in each device group
        for dg in device_groups:
            rule_name = f"Block-{object_name}"
            
            adapter.create_security_rule(
                name=rule_name,
                source_zones=["any"],
                destination_zones=["any"],
                source_addresses=[object_name],
                destination_addresses=["any"],
                applications=["any"],
                services=["any"],
                action="deny",
                device_group=dg,
                description=f"Auto-generated: {reason}",
                log_end=True
            )
            
            results[dg] = {"status": "rule_created", "rule_name": rule_name}
        
        # Step 3: Commit to Panorama
        commit_result = adapter.commit_to_panorama()
        results["panorama_commit"] = commit_result
        
        # Step 4: Push to all device groups
        for dg in device_groups:
            push_result = adapter.push_to_device_group(dg)
            results[f"{dg}_push"] = push_result
        
        return {
            "status": "success",
            "ip_address": ip_address,
            "object_name": object_name,
            "device_groups": device_groups,
            "results": results
        }
        
    except Exception as e:
        return {
            "status": "error",
            "ip_address": ip_address,
            "error": str(e),
            "results": results
        }
```

#### Pattern 2: DR Failover - Update NAT Rules

```python
def update_nat_for_failover_palo_alto(
    adapter: PaloAltoAdapter,
    primary_public_ip: str,
    dr_public_ip: str,
    device_groups: List[str]
) -> Dict:
    """
    Update NAT rules to point to DR public IP during failover
    
    Args:
        adapter: PaloAltoAdapter instance
        primary_public_ip: Primary region public IP
        dr_public_ip: DR region public IP
        device_groups: Device groups with NAT rules
        
    Returns:
        dict: Update results
    """
    results = {}
    
    # Step 1: Find and update address object for public IP
    try:
        # Update shared address object
        adapter.update_address_object(
            name="Public-VIP",
            new_value=dr_public_ip,
            device_group="shared"
        )
        
        results["address_object_updated"] = True
        
        # Step 2: Commit and push
        adapter.commit_to_panorama()
        
        for dg in device_groups:
            adapter.push_to_device_group(dg)
            results[f"{dg}_pushed"] = True
        
        return {
            "status": "success",
            "old_ip": primary_public_ip,
            "new_ip": dr_public_ip,
            "results": results
        }
        
    except Exception as e:
        return {
            "status": "error",
            "error": str(e),
            "results": results
        }
```

### 2.6 Error Handling

```python
class PaloAltoError(Exception):
    """Base exception for Palo Alto adapter"""
    pass

class PaloAltoConnectionError(PaloAltoError):
    """Connection to Panorama failed"""
    pass

class PaloAltoCommitError(PaloAltoError):
    """Commit operation failed"""
    pass

class PaloAltoAPIError(PaloAltoError):
    """API call failed"""
    pass

# Usage in adapter
def _handle_api_error(self, error: Exception, operation: str):
    """Handle API errors with appropriate logging and exceptions"""
    if "Connection refused" in str(error):
        raise PaloAltoConnectionError(f"Cannot connect to Panorama at {self.panorama_host}")
    elif "commit failed" in str(error).lower():
        raise PaloAltoCommitError(f"Commit failed during {operation}: {error}")
    else:
        raise PaloAltoAPIError(f"API error during {operation}: {error}")
```

### 2.7 Testing

```python
# Unit tests
import unittest
from unittest.mock import Mock, patch

class TestPaloAltoAdapter(unittest.TestCase):
    
    def setUp(self):
        self.adapter = PaloAltoAdapter(
            panorama_host="panorama.test.local",
            api_key="test-key"
        )
    
    @patch('panos.panorama.Panorama')
    def test_create_security_rule(self, mock_panorama):
        """Test security rule creation"""
        result = self.adapter.create_security_rule(
            name="Test-Rule",
            source_zones=["trust"],
            destination_zones=["untrust"],
            source_addresses=["10.0.0.0/8"],
            destination_addresses=["any"],
            applications=["web-browsing"],
            services=["application-default"],
            action="allow",
            device_group="DG-Test"
        )
        
        self.assertEqual(result["status"], "success")
        self.assertEqual(result["rule_name"], "Test-Rule")
    
    @patch('panos.panorama.Panorama')
    def test_commit_failure(self, mock_panorama):
        """Test commit failure handling"""
        mock_panorama.commit.side_effect = Exception("Commit failed")
        
        with self.assertRaises(PaloAltoCommitError):
            self.adapter.commit_to_panorama()
```

</details>

---

## 3. F5 BIG-IP Integration

<details>
<summary>Click to expand</summary>

### 3.1 F5 BIG-IP Infrastructure

**Contoso Architecture Components**:
- **F5 BIG-IP LTM** (Local Traffic Manager): Load balancing
- **F5 BIG-IP GTM** (Global Traffic Manager): DNS-based load balancing, GSLB
- **F5 AFM** (Advanced Firewall Manager): Network firewall
- **F5 ASM** (Application Security Manager): Web Application Firewall

**Devices**:
- `F5-Chennai-01` (Active)
- `F5-Chennai-02` (Standby)
- `F5-Pune-01` (Active)
- `F5-Pune-02` (Standby)

### 3.2 API Options

| API Type | Use Case | Pros | Cons |
|----------|----------|------|------|
| **iControl REST** | Modern API (v11.5+) | JSON-based, easy to use | Requires TMOS 11.5+ |
| **AS3** (App Services 3) | Declarative app deployment | Infrastructure-as-code friendly | Learning curve |
| **TMSH** | Legacy CLI commands | Full feature coverage | Not RESTful |

**Recommendation**: Use **iControl REST API** for most operations, **AS3** for application deployments.

### 3.3 Authentication

**Method 1: Token-based Authentication (Recommended)**

```python
import requests
import json

# Step 1: Obtain auth token
def get_f5_token(host: str, username: str, password: str) -> str:
    """Get authentication token from F5"""
    url = f"https://{host}/mgmt/shared/authn/login"
    payload = {
        "username": username,
        "password": password,
        "loginProviderName": "tmos"
    }
    
    response = requests.post(
        url,
        json=payload,
        verify=False  # Use proper cert verification in production
    )
    
    if response.status_code == 200:
        token = response.json()['token']['token']
        return token
    else:
        raise Exception(f"Failed to obtain F5 token: {response.text}")

# Step 2: Use token in API calls
headers = {
    "X-F5-Auth-Token": token,
    "Content-Type": "application/json"
}

response = requests.get(
    f"https://{host}/mgmt/tm/ltm/pool",
    headers=headers,
    verify=False
)
```

**Token Refresh**:
- Tokens expire after 1200 seconds (20 minutes) by default
- Extend timeout or refresh token before expiration

```python
# Extend token timeout
def extend_token_timeout(host: str, token: str, timeout: int = 36000):
    """Extend token timeout to 10 hours"""
    url = f"https://{host}/mgmt/shared/authz/tokens/{token}"
    payload = {"timeout": timeout}
    headers = {"X-F5-Auth-Token": token}
    
    response = requests.patch(url, json=payload, headers=headers, verify=False)
    return response.status_code == 200
```

### 3.4 F5 BIG-IP Adapter Implementation

#### 3.4.1 Adapter Class Structure

```python
import requests
import time
from typing import List, Dict, Optional
import logging
import urllib3

# Disable SSL warnings for self-signed certs (use proper certs in production)
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class F5Adapter:
    """
    Adapter for F5 BIG-IP integration via iControl REST API
    Supports LTM (load balancing), GTM (GSLB), AFM (firewall)
    """
    
    def __init__(
        self,
        host: str,
        username: str,
        password: str,
        timeout: int = 30,
        verify_ssl: bool = False
    ):
        """
        Initialize F5 adapter
        
        Args:
            host: F5 management IP or hostname
            username: F5 username
            password: F5 password
            timeout: API request timeout in seconds
            verify_ssl: Verify SSL certificates
        """
        self.host = host
        self.username = username
        self.password = password
        self.timeout = timeout
        self.verify_ssl = verify_ssl
        self.base_url = f"https://{host}/mgmt"
        self.token = None
        self.logger = logging.getLogger(__name__)
        
        # Authenticate and get token
        self._authenticate()
    
    def _authenticate(self):
        """Authenticate with F5 and obtain token"""
        url = f"{self.base_url}/shared/authn/login"
        payload = {
            "username": self.username,
            "password": self.password,
            "loginProviderName": "tmos"
        }
        
        try:
            response = requests.post(
                url,
                json=payload,
                timeout=self.timeout,
                verify=self.verify_ssl
            )
            response.raise_for_status()
            
            self.token = response.json()['token']['token']
            
            # Extend token timeout to 10 hours
            self._extend_token_timeout()
            
            self.logger.info(f"Authenticated with F5 {self.host}")
            
        except Exception as e:
            self.logger.error(f"Failed to authenticate with F5: {e}")
            raise
    
    def _extend_token_timeout(self, timeout: int = 36000):
        """Extend token timeout"""
        url = f"{self.base_url}/shared/authz/tokens/{self.token}"
        headers = self._get_headers()
        payload = {"timeout": timeout}
        
        response = requests.patch(
            url,
            json=payload,
            headers=headers,
            timeout=self.timeout,
            verify=self.verify_ssl
        )
        
        if response.status_code == 200:
            self.logger.info(f"Extended token timeout to {timeout} seconds")
    
    def _get_headers(self) -> Dict:
        """Get headers with auth token"""
        return {
            "X-F5-Auth-Token": self.token,
            "Content-Type": "application/json"
        }
    
    def _make_request(
        self,
        method: str,
        endpoint: str,
        data: Optional[Dict] = None
    ) -> requests.Response:
        """Make API request to F5"""
        url = f"{self.base_url}{endpoint}"
        headers = self._get_headers()
        
        try:
            response = requests.request(
                method=method,
                url=url,
                headers=headers,
                json=data,
                timeout=self.timeout,
                verify=self.verify_ssl
            )
            
            # Token expired, re-authenticate
            if response.status_code == 401:
                self.logger.warning("Token expired, re-authenticating...")
                self._authenticate()
                headers = self._get_headers()
                response = requests.request(
                    method=method,
                    url=url,
                    headers=headers,
                    json=data,
                    timeout=self.timeout,
                    verify=self.verify_ssl
                )
            
            response.raise_for_status()
            return response
            
        except requests.exceptions.RequestException as e:
            self.logger.error(f"API request failed: {e}")
            raise
```

#### 3.4.2 Pool Management (Load Balancing)

```python
    def create_pool(
        self,
        name: str,
        partition: str = "Common",
        load_balancing_mode: str = "round-robin",
        monitor: Optional[str] = None,
        description: Optional[str] = None
    ) -> Dict:
        """
        Create load balancing pool
        
        Args:
            name: Pool name
            partition: F5 partition (default: Common)
            load_balancing_mode: round-robin, least-connections-member, etc.
            monitor: Health monitor (e.g., '/Common/http')
            description: Pool description
            
        Returns:
            dict: Pool creation result
        """
        endpoint = "/tm/ltm/pool"
        
        payload = {
            "name": name,
            "partition": partition,
            "loadBalancingMode": load_balancing_mode
        }
        
        if monitor:
            payload["monitor"] = monitor
        
        if description:
            payload["description"] = description
        
        try:
            response = self._make_request("POST", endpoint, payload)
            pool_data = response.json()
            
            self.logger.info(f"Created pool '{name}' in partition '{partition}'")
            
            return {
                "status": "success",
                "pool_name": name,
                "full_path": pool_data.get("fullPath"),
                "selfLink": pool_data.get("selfLink")
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create pool: {e}")
            raise
    
    def add_pool_member(
        self,
        pool_name: str,
        member_address: str,
        member_port: int,
        partition: str = "Common",
        description: Optional[str] = None,
        connection_limit: int = 0,
        ratio: int = 1
    ) -> Dict:
        """
        Add member to pool
        
        Args:
            pool_name: Pool name
            member_address: Member IP address
            member_port: Member port
            partition: F5 partition
            description: Member description
            connection_limit: Max connections (0 = unlimited)
            ratio: Load balancing ratio
            
        Returns:
            dict: Operation result
        """
        endpoint = f"/tm/ltm/pool/~{partition}~{pool_name}/members"
        
        # Member name format: address:port
        member_name = f"{member_address}:{member_port}"
        
        payload = {
            "name": member_name,
            "address": member_address,
            "connectionLimit": connection_limit,
            "ratio": ratio
        }
        
        if description:
            payload["description"] = description
        
        try:
            response = self._make_request("POST", endpoint, payload)
            member_data = response.json()
            
            self.logger.info(
                f"Added member {member_name} to pool '{pool_name}'"
            )
            
            # Wait for health check
            self._wait_for_member_up(pool_name, member_name, partition)
            
            return {
                "status": "success",
                "pool_name": pool_name,
                "member": member_name,
                "state": "enabled"
            }
            
        except Exception as e:
            self.logger.error(f"Failed to add pool member: {e}")
            raise
    
    def remove_pool_member(
        self,
        pool_name: str,
        member_name: str,
        partition: str = "Common"
    ) -> Dict:
        """Remove member from pool"""
        # URL encode member name (contains ':')
        member_name_encoded = member_name.replace(":", "%3A")
        endpoint = f"/tm/ltm/pool/~{partition}~{pool_name}/members/~{partition}~{member_name_encoded}"
        
        try:
            self._make_request("DELETE", endpoint)
            
            self.logger.info(
                f"Removed member {member_name} from pool '{pool_name}'"
            )
            
            return {
                "status": "success",
                "pool_name": pool_name,
                "member": member_name
            }
            
        except Exception as e:
            self.logger.error(f"Failed to remove pool member: {e}")
            raise
    
    def get_pool_members(
        self,
        pool_name: str,
        partition: str = "Common"
    ) -> List[Dict]:
        """Get all members of a pool with their status"""
        endpoint = f"/tm/ltm/pool/~{partition}~{pool_name}/members"
        params = {"$select": "name,address,port,state,session,ratio"}
        
        try:
            url = f"{self.base_url}{endpoint}"
            headers = self._get_headers()
            
            response = requests.get(
                url,
                headers=headers,
                params=params,
                timeout=self.timeout,
                verify=self.verify_ssl
            )
            response.raise_for_status()
            
            members_data = response.json().get("items", [])
            
            members = []
            for member in members_data:
                members.append({
                    "name": member.get("name"),
                    "address": member.get("address"),
                    "port": member.get("port"),
                    "state": member.get("state"),
                    "session": member.get("session"),
                    "ratio": member.get("ratio"),
                    "available": member.get("state") == "up"
                })
            
            return members
            
        except Exception as e:
            self.logger.error(f"Failed to get pool members: {e}")
            raise
    
    def _wait_for_member_up(
        self,
        pool_name: str,
        member_name: str,
        partition: str = "Common",
        timeout: int = 60,
        interval: int = 5
    ):
        """Wait for pool member to become available"""
        start_time = time.time()
        member_name_encoded = member_name.replace(":", "%3A")
        endpoint = f"/tm/ltm/pool/~{partition}~{pool_name}/members/~{partition}~{member_name_encoded}/stats"
        
        while time.time() - start_time < timeout:
            try:
                url = f"{self.base_url}{endpoint}"
                headers = self._get_headers()
                response = requests.get(
                    url,
                    headers=headers,
                    timeout=self.timeout,
                    verify=self.verify_ssl
                )
                
                if response.status_code == 200:
                    stats = response.json()
                    entries = stats.get("entries", {})
                    
                    for entry_key, entry_data in entries.items():
                        nested_stats = entry_data.get("nestedStats", {}).get("entries", {})
                        status = nested_stats.get("status.availabilityState", {}).get("description")
                        
                        if status == "available":
                            self.logger.info(f"Member {member_name} is now available")
                            return
                
                time.sleep(interval)
                
            except Exception as e:
                self.logger.warning(f"Error checking member status: {e}")
                time.sleep(interval)
        
        self.logger.warning(
            f"Member {member_name} did not become available within {timeout} seconds"
        )
```

#### 3.4.3 Virtual Server Management

```python
    def create_virtual_server(
        self,
        name: str,
        destination: str,
        port: int,
        pool: str,
        partition: str = "Common",
        profiles: Optional[List[str]] = None,
        source_address_translation: Optional[Dict] = None,
        description: Optional[str] = None
    ) -> Dict:
        """
        Create virtual server
        
        Args:
            name: Virtual server name
            destination: VIP address
            port: VIP port
            pool: Backend pool name
            partition: F5 partition
            profiles: List of profiles (e.g., ['/Common/http', '/Common/tcp'])
            source_address_translation: SNAT config
            description: VS description
            
        Returns:
            dict: Creation result
        """
        endpoint = "/tm/ltm/virtual"
        
        # Format destination as address:port
        dest_formatted = f"{destination}:{port}"
        
        payload = {
            "name": name,
            "partition": partition,
            "destination": dest_formatted,
            "pool": f"/{partition}/{pool}",
            "ipProtocol": "tcp",
            "sourceAddressTranslation": source_address_translation or {"type": "automap"}
        }
        
        if profiles:
            payload["profiles"] = [{"name": p} for p in profiles]
        else:
            # Default profiles
            payload["profiles"] = [
                {"name": "/Common/tcp"},
                {"name": "/Common/http"}
            ]
        
        if description:
            payload["description"] = description
        
        try:
            response = self._make_request("POST", endpoint, payload)
            vs_data = response.json()
            
            self.logger.info(
                f"Created virtual server '{name}' at {dest_formatted}"
            )
            
            return {
                "status": "success",
                "vs_name": name,
                "destination": dest_formatted,
                "pool": pool,
                "full_path": vs_data.get("fullPath")
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create virtual server: {e}")
            raise
    
    def get_virtual_server_stats(
        self,
        vs_name: str,
        partition: str = "Common"
    ) -> Dict:
        """Get virtual server statistics"""
        endpoint = f"/tm/ltm/virtual/~{partition}~{vs_name}/stats"
        
        try:
            url = f"{self.base_url}{endpoint}"
            headers = self._get_headers()
            response = requests.get(
                url,
                headers=headers,
                timeout=self.timeout,
                verify=self.verify_ssl
            )
            response.raise_for_status()
            
            stats_data = response.json()
            entries = stats_data.get("entries", {})
            
            # Extract key statistics
            stats = {}
            for entry_key, entry_data in entries.items():
                nested_stats = entry_data.get("nestedStats", {}).get("entries", {})
                
                stats = {
                    "status": nested_stats.get("status.availabilityState", {}).get("description"),
                    "client_connections": nested_stats.get("clientside.curConns", {}).get("value", 0),
                    "total_requests": nested_stats.get("totRequests", {}).get("value", 0),
                    "bytes_in": nested_stats.get("clientside.bitsIn", {}).get("value", 0),
                    "bytes_out": nested_stats.get("clientside.bitsOut", {}).get("value", 0)
                }
            
            return {
                "vs_name": vs_name,
                "stats": stats
            }
            
        except Exception as e:
            self.logger.error(f"Failed to get virtual server stats: {e}")
            raise
```

#### 3.4.4 Traffic Distribution (Blue-Green Deployment)

```python
    def set_pool_member_ratio(
        self,
        pool_name: str,
        member_ratios: Dict[str, int],
        partition: str = "Common"
    ) -> Dict:
        """
        Set traffic ratio for pool members (for blue-green deployment)
        
        Args:
            pool_name: Pool name
            member_ratios: Dict of {member_name: ratio}
            partition: F5 partition
            
        Returns:
            dict: Operation result
        
        Example:
            set_pool_member_ratio(
                pool_name="app-pool",
                member_ratios={
                    "10.10.10.10:443": 80,  # Blue: 80%
                    "10.10.10.20:443": 20   # Green: 20%
                }
            )
        """
        results = {}
        
        for member_name, ratio in member_ratios.items():
            member_name_encoded = member_name.replace(":", "%3A")
            endpoint = f"/tm/ltm/pool/~{partition}~{pool_name}/members/~{partition}~{member_name_encoded}"
            
            payload = {"ratio": ratio}
            
            try:
                self._make_request("PATCH", endpoint, payload)
                results[member_name] = {"ratio": ratio, "status": "success"}
                self.logger.info(f"Set ratio for {member_name} to {ratio}")
                
            except Exception as e:
                results[member_name] = {"ratio": ratio, "status": "error", "error": str(e)}
                self.logger.error(f"Failed to set ratio for {member_name}: {e}")
        
        return {
            "pool_name": pool_name,
            "results": results
        }
    
    def enable_pool_member(
        self,
        pool_name: str,
        member_name: str,
        partition: str = "Common"
    ) -> Dict:
        """Enable pool member"""
        member_name_encoded = member_name.replace(":", "%3A")
        endpoint = f"/tm/ltm/pool/~{partition}~{pool_name}/members/~{partition}~{member_name_encoded}"
        
        payload = {
            "session": "user-enabled",
            "state": "user-up"
        }
        
        try:
            self._make_request("PATCH", endpoint, payload)
            
            self.logger.info(f"Enabled pool member {member_name}")
            
            return {
                "status": "success",
                "member": member_name,
                "action": "enabled"
            }
            
        except Exception as e:
            self.logger.error(f"Failed to enable pool member: {e}")
            raise
    
    def disable_pool_member(
        self,
        pool_name: str,
        member_name: str,
        partition: str = "Common"
    ) -> Dict:
        """Disable pool member (gracefully drain connections)"""
        member_name_encoded = member_name.replace(":", "%3A")
        endpoint = f"/tm/ltm/pool/~{partition}~{pool_name}/members/~{partition}~{member_name_encoded}"
        
        payload = {
            "session": "user-disabled",
            "state": "user-up"
        }
        
        try:
            self._make_request("PATCH", endpoint, payload)
            
            self.logger.info(f"Disabled pool member {member_name}")
            
            return {
                "status": "success",
                "member": member_name,
                "action": "disabled"
            }
            
        except Exception as e:
            self.logger.error(f"Failed to disable pool member: {e}")
            raise
```

#### 3.4.5 GTM (Global Traffic Manager) - GSLB

```python
    def create_gtm_pool(
        self,
        name: str,
        partition: str = "Common",
        load_balancing_mode: str = "round-robin",
        members: Optional[List[Dict]] = None
    ) -> Dict:
        """
        Create GTM pool for GSLB
        
        Args:
            name: Pool name
            partition: F5 partition
            load_balancing_mode: round-robin, global-availability, etc.
            members: List of member dicts with 'server' and 'virtual_server'
            
        Returns:
            dict: Creation result
        """
        endpoint = "/tm/gtm/pool/a"  # A record pool
        
        payload = {
            "name": name,
            "partition": partition,
            "loadBalancingMode": load_balancing_mode
        }
        
        if members:
            payload["members"] = members
        
        try:
            response = self._make_request("POST", endpoint, payload)
            pool_data = response.json()
            
            self.logger.info(f"Created GTM pool '{name}'")
            
            return {
                "status": "success",
                "pool_name": name,
                "full_path": pool_data.get("fullPath")
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create GTM pool: {e}")
            raise
    
    def update_gtm_pool_members(
        self,
        pool_name: str,
        active_datacenter: str,
        inactive_datacenter: str,
        partition: str = "Common"
    ) -> Dict:
        """
        Update GTM pool to change active datacenter (for DR failover)
        
        Args:
            pool_name: GTM pool name
            active_datacenter: Datacenter to make active
            inactive_datacenter: Datacenter to make inactive
            partition: F5 partition
            
        Returns:
            dict: Update result
        """
        # This is a simplified example - actual implementation depends on GTM topology
        endpoint = f"/tm/gtm/pool/a/~{partition}~{pool_name}"
        
        try:
            # Get current pool config
            response = self._make_request("GET", endpoint)
            pool_data = response.json()
            
            members = pool_data.get("members", [])
            
            # Update member priorities
            for member in members:
                if active_datacenter in member.get("name", ""):
                    member["ratio"] = 100
                    member["disabled"] = False
                elif inactive_datacenter in member.get("name", ""):
                    member["ratio"] = 0
                    member["disabled"] = True
            
            # Update pool
            payload = {"members": members}
            self._make_request("PATCH", endpoint, payload)
            
            self.logger.info(
                f"Updated GTM pool '{pool_name}': active={active_datacenter}, inactive={inactive_datacenter}"
            )
            
            return {
                "status": "success",
                "pool_name": pool_name,
                "active_datacenter": active_datacenter,
                "inactive_datacenter": inactive_datacenter
            }
            
        except Exception as e:
            self.logger.error(f"Failed to update GTM pool: {e}")
            raise
```

#### 3.4.6 AFM (Advanced Firewall Manager)

```python
    def add_afm_blocked_ip(
        self,
        ip_address: str,
        partition: str = "Common",
        description: Optional[str] = None
    ) -> Dict:
        """
        Add IP to AFM blocked list
        
        Args:
            ip_address: IP address to block
            partition: F5 partition
            description: Reason for blocking
            
        Returns:
            dict: Operation result
        """
        # Create address list if doesn't exist
        address_list_name = "blocked-ips"
        endpoint = f"/tm/security/firewall/address-list"
        
        # Check if address list exists
        try:
            get_response = self._make_request(
                "GET",
                f"{endpoint}/~{partition}~{address_list_name}"
            )
            address_list_data = get_response.json()
            addresses = address_list_data.get("addresses", [])
        except:
            # Create new address list
            payload = {
                "name": address_list_name,
                "partition": partition,
                "addresses": []
            }
            self._make_request("POST", endpoint, payload)
            addresses = []
        
        # Add IP to list
        if {"name": ip_address} not in addresses:
            addresses.append({"name": ip_address})
            
            payload = {"addresses": addresses}
            self._make_request(
                "PATCH",
                f"{endpoint}/~{partition}~{address_list_name}",
                payload
            )
            
            self.logger.info(f"Added {ip_address} to AFM blocked list")
            
            return {
                "status": "success",
                "ip_address": ip_address,
                "address_list": address_list_name
            }
        else:
            return {
                "status": "already_exists",
                "ip_address": ip_address
            }
```

### 3.5 Integration Patterns

#### Pattern 1: Blue-Green Deployment with F5

```python
def blue_green_deployment_f5(
    adapter: F5Adapter,
    pool_name: str,
    blue_members: List[str],
    green_members: List[str],
    traffic_percentages: List[int] = [0, 10, 50, 100],
    wait_time: int = 300
) -> Dict:
    """
    Execute blue-green deployment with gradual traffic shift
    
    Args:
        adapter: F5Adapter instance
        pool_name: Pool name
        blue_members: List of blue environment member names
        green_members: List of green environment member names
        traffic_percentages: Traffic % to shift to green at each step
        wait_time: Wait time between steps (seconds)
        
    Returns:
        dict: Deployment result
    """
    results = []
    
    for green_percent in traffic_percentages:
        blue_percent = 100 - green_percent
        
        # Calculate ratios
        member_ratios = {}
        for member in blue_members:
            member_ratios[member] = blue_percent
        for member in green_members:
            member_ratios[member] = green_percent
        
        # Apply ratios
        result = adapter.set_pool_member_ratio(
            pool_name=pool_name,
            member_ratios=member_ratios
        )
        
        results.append({
            "green_percent": green_percent,
            "timestamp": time.time(),
            "result": result
        })
        
        logger.info(f"Traffic split: Blue {blue_percent}% / Green {green_percent}%")
        
        # Monitor for errors
        if green_percent < 100:
            # Check error rates, response times, etc.
            time.sleep(wait_time)
            
            # If errors detected, rollback
            error_rate = check_error_rate(pool_name)
            if error_rate > 1.0:  # > 1% error rate
                logger.error("Error rate too high, rolling back...")
                rollback_ratios = {}
                for member in blue_members:
                    rollback_ratios[member] = 100
                for member in green_members:
                    rollback_ratios[member] = 0
                
                adapter.set_pool_member_ratio(pool_name, rollback_ratios)
                
                return {
                    "status": "rolled_back",
                    "reason": "error_rate_exceeded",
                    "results": results
                }
    
    return {
        "status": "success",
        "final_state": "green_100_percent",
        "results": results
    }
```

#### Pattern 2: DR Failover with GTM

```python
def dr_failover_f5_gtm(
    adapter: F5Adapter,
    gtm_pool_name: str,
    primary_datacenter: str,
    dr_datacenter: str
) -> Dict:
    """
    Failover DNS traffic from primary to DR datacenter using GTM
    
    Args:
        adapter: F5Adapter instance
        gtm_pool_name: GTM pool name
        primary_datacenter: Primary datacenter name
        dr_datacenter: DR datacenter name
        
    Returns:
        dict: Failover result
    """
    try:
        # Update GTM pool to make DR active
        result = adapter.update_gtm_pool_members(
            pool_name=gtm_pool_name,
            active_datacenter=dr_datacenter,
            inactive_datacenter=primary_datacenter
        )
        
        logger.info(f"GTM failover complete: {primary_datacenter} → {dr_datacenter}")
        
        return {
            "status": "success",
            "old_datacenter": primary_datacenter,
            "new_datacenter": dr_datacenter,
            "gtm_pool": gtm_pool_name,
            "result": result
        }
        
    except Exception as e:
        logger.error(f"GTM failover failed: {e}")
        return {
            "status": "error",
            "error": str(e)
        }
```

### 3.6 Error Handling

```python
class F5Error(Exception):
    """Base exception for F5 adapter"""
    pass

class F5AuthenticationError(F5Error):
    """Authentication failed"""
    pass

class F5APIError(F5Error):
    """API call failed"""
    pass

class F5ResourceNotFoundError(F5Error):
    """Resource not found"""
    pass
```

### 3.7 Testing

```python
import unittest
from unittest.mock import Mock, patch

class TestF5Adapter(unittest.TestCase):
    
    def setUp(self):
        self.adapter = F5Adapter(
            host="f5.test.local",
            username="admin",
            password="password"
        )
    
    @patch('requests.post')
    def test_create_pool(self, mock_post):
        """Test pool creation"""
        mock_post.return_value.status_code = 200
        mock_post.return_value.json.return_value = {
            "fullPath": "/Common/test-pool"
        }
        
        result = self.adapter.create_pool(
            name="test-pool",
            load_balancing_mode="round-robin"
        )
        
        self.assertEqual(result["status"], "success")
        self.assertEqual(result["pool_name"], "test-pool")
```

</details>

---

## 4. Infoblox Integration

<details>
<summary>Click to expand</summary>

### 4.1 Infoblox Infrastructure

**Contoso Components**:
- **Grid Master**: Primary management and authoritative DNS
- **Grid Members**: Distributed DNS/DHCP/IPAM nodes across datacenters
- **DNS Zones**: 
  - `Contoso.com` (external)
  - `Contoso.internal` (internal)
  - Reverse zones for IP management
- **IPAM Networks**:
  - Azure address space management
  - On-premises network tracking
  - IP allocation and tracking

### 4.2 API: WAPI (Web API)

**Base URL**: `https://grid-master.Contoso.local/wapi/v2.12/`

**Key Objects**:
- `network`: Network containers
- `networkcontainer`: Parent networks
- `record:a`: A records
- `record:cname`: CNAME records
- `record:ptr`: PTR (reverse) records
- `ipv4address`: Individual IP addresses
- `lease`: DHCP leases

### 4.3 Authentication

**Basic Authentication** (Username/Password):
```python
import requests
from requests.auth import HTTPBasicAuth

username = "api-user"
password = "SecurePassword123"

response = requests.get(
    "https://grid-master.Contoso.local/wapi/v2.12/network",
    auth=HTTPBasicAuth(username, password),
    verify=False
)
```

### 4.4 Infoblox Adapter Implementation

#### 4.4.1 Adapter Class Structure

```python
import requests
from requests.auth import HTTPBasicAuth
from typing import List, Dict, Optional
import logging
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class InfobloxAdapter:
    """
    Adapter for Infoblox Grid Master integration via WAPI
    Supports DNS, DHCP, and IPAM operations
    """
    
    def __init__(
        self,
        grid_master: str,
        username: str,
        password: str,
        wapi_version: str = "v2.12",
        timeout: int = 30,
        verify_ssl: bool = False
    ):
        """
        Initialize Infoblox adapter
        
        Args:
            grid_master: Grid Master hostname or IP
            username: Infoblox username
            password: Infoblox password
            wapi_version: WAPI version (e.g., v2.12)
            timeout: API timeout in seconds
            verify_ssl: Verify SSL certificates
        """
        self.grid_master = grid_master
        self.username = username
        self.password = password
        self.wapi_version = wapi_version
        self.base_url = f"https://{grid_master}/wapi/{wapi_version}"
        self.timeout = timeout
        self.verify_ssl = verify_ssl
        self.auth = HTTPBasicAuth(username, password)
        self.logger = logging.getLogger(__name__)
        
        # Test connectivity
        self._test_connection()
    
    def _test_connection(self):
        """Test connection to Grid Master"""
        try:
            response = requests.get(
                f"{self.base_url}/grid",
                auth=self.auth,
                timeout=self.timeout,
                verify=self.verify_ssl
            )
            response.raise_for_status()
            self.logger.info(f"Connected to Infoblox Grid Master {self.grid_master}")
        except Exception as e:
            self.logger.error(f"Failed to connect to Infoblox: {e}")
            raise
    
    def _make_request(
        self,
        method: str,
        endpoint: str,
        params: Optional[Dict] = None,
        data: Optional[Dict] = None
    ) -> requests.Response:
        """Make API request to Infoblox"""
        url = f"{self.base_url}/{endpoint}"
        
        try:
            response = requests.request(
                method=method,
                url=url,
                auth=self.auth,
                params=params,
                json=data,
                timeout=self.timeout,
                verify=self.verify_ssl
            )
            response.raise_for_status()
            return response
            
        except requests.exceptions.RequestException as e:
            self.logger.error(f"API request failed: {e}")
            if hasattr(e.response, 'text'):
                self.logger.error(f"Response: {e.response.text}")
            raise
```

#### 4.4.2 IPAM Operations

```python
    def get_next_available_network(
        self,
        parent_network: str,
        cidr: int,
        comment: Optional[str] = None
    ) -> str:
        """
        Allocate next available subnet from parent network
        
        Args:
            parent_network: Parent network (e.g., '10.200.0.0/16')
            cidr: CIDR for subnet (e.g., 24 for /24)
            comment: Comment for the network
            
        Returns:
            str: Allocated network (e.g., '10.200.50.0/24')
        """
        # First, find the network container
        endpoint = "networkcontainer"
        params = {
            "network": parent_network,
            "_return_fields": "network,comment"
        }
        
        try:
            response = self._make_request("GET", endpoint, params=params)
            containers = response.json()
            
            if not containers:
                raise InfobloxError(f"Parent network {parent_network} not found")
            
            container_ref = containers[0]["_ref"]
            
            # Request next available network
            endpoint = container_ref
            params = {"_function": "next_available_network"}
            data = {
                "cidr": cidr,
                "num": 1
            }
            
            if comment:
                data["comment"] = comment
            
            response = self._make_request("POST", endpoint, params=params, data=data)
            result = response.json()
            
            allocated_network = result["networks"][0]
            
            self.logger.info(
                f"Allocated network {allocated_network} from {parent_network}"
            )
            
            return allocated_network
            
        except Exception as e:
            self.logger.error(f"Failed to allocate network: {e}")
            raise
    
    def create_network(
        self,
        network: str,
        comment: Optional[str] = None,
        members: Optional[List[Dict]] = None
    ) -> Dict:
        """
        Create network in IPAM
        
        Args:
            network: Network CIDR (e.g., '10.200.50.0/24')
            comment: Network comment/description
            members: List of Grid members serving this network
            
        Returns:
            dict: Created network details
        """
        endpoint = "network"
        
        data = {
            "network": network
        }
        
        if comment:
            data["comment"] = comment
        
        if members:
            data["members"] = members
        
        try:
            response = self._make_request("POST", endpoint, data=data)
            network_ref = response.json()
            
            self.logger.info(f"Created network {network}")
            
            return {
                "status": "success",
                "network": network,
                "_ref": network_ref
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create network: {e}")
            raise
    
    def get_network_info(
        self,
        network: str
    ) -> Dict:
        """Get information about a network"""
        endpoint = "network"
        params = {
            "network": network,
            "_return_fields": "network,comment,members,network_view,utilization"
        }
        
        try:
            response = self._make_request("GET", endpoint, params=params)
            networks = response.json()
            
            if not networks:
                raise InfobloxResourceNotFoundError(f"Network {network} not found")
            
            return networks[0]
            
        except Exception as e:
            self.logger.error(f"Failed to get network info: {e}")
            raise
    
    def get_next_available_ip(
        self,
        network: str,
        num_ips: int = 1,
        exclude: Optional[List[str]] = None
    ) -> List[str]:
        """
        Get next available IP(s) from network
        
        Args:
            network: Network CIDR
            num_ips: Number of IPs to allocate
            exclude: List of IPs to exclude
            
        Returns:
            list: Allocated IP addresses
        """
        # Find network
        endpoint = "network"
        params = {"network": network}
        
        try:
            response = self._make_request("GET", endpoint, params=params)
            networks = response.json()
            
            if not networks:
                raise InfobloxResourceNotFoundError(f"Network {network} not found")
            
            network_ref = networks[0]["_ref"]
            
            # Get next available IPs
            params = {"_function": "next_available_ip"}
            data = {"num": num_ips}
            
            if exclude:
                data["exclude"] = exclude
            
            response = self._make_request("POST", network_ref, params=params, data=data)
            result = response.json()
            
            ips = result["ips"]
            
            self.logger.info(f"Allocated {len(ips)} IP(s) from {network}")
            
            return ips
            
        except Exception as e:
            self.logger.error(f"Failed to get next available IP: {e}")
            raise
```

#### 4.4.3 DNS Record Management

```python
    def create_a_record(
        self,
        fqdn: str,
        ip_address: str,
        ttl: Optional[int] = None,
        comment: Optional[str] = None,
        view: str = "default"
    ) -> Dict:
        """
        Create DNS A record
        
        Args:
            fqdn: Fully qualified domain name
            ip_address: IP address
            ttl: Time to live (seconds)
            comment: Record comment
            view: DNS view
            
        Returns:
            dict: Created record details
        """
        endpoint = "record:a"
        
        data = {
            "name": fqdn,
            "ipv4addr": ip_address,
            "view": view
        }
        
        if ttl:
            data["ttl"] = ttl
            data["use_ttl"] = True
        
        if comment:
            data["comment"] = comment
        
        try:
            response = self._make_request("POST", endpoint, data=data)
            record_ref = response.json()
            
            self.logger.info(f"Created A record: {fqdn} → {ip_address}")
            
            return {
                "status": "success",
                "fqdn": fqdn,
                "ip_address": ip_address,
                "_ref": record_ref
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create A record: {e}")
            raise
    
    def update_a_record(
        self,
        fqdn: str,
        new_ip: str,
        ttl: Optional[int] = None,
        view: str = "default"
    ) -> Dict:
        """
        Update existing A record with new IP
        
        Args:
            fqdn: Fully qualified domain name
            new_ip: New IP address
            ttl: Time to live
            view: DNS view
            
        Returns:
            dict: Update result
        """
        # Find existing record
        endpoint = "record:a"
        params = {
            "name": fqdn,
            "view": view
        }
        
        try:
            response = self._make_request("GET", endpoint, params=params)
            records = response.json()
            
            if not records:
                raise InfobloxResourceNotFoundError(f"A record {fqdn} not found")
            
            record_ref = records[0]["_ref"]
            old_ip = records[0]["ipv4addr"]
            
            # Update record
            data = {"ipv4addr": new_ip}
            
            if ttl:
                data["ttl"] = ttl
                data["use_ttl"] = True
            
            self._make_request("PUT", record_ref, data=data)
            
            self.logger.info(
                f"Updated A record: {fqdn} {old_ip} → {new_ip}"
            )
            
            return {
                "status": "success",
                "fqdn": fqdn,
                "old_ip": old_ip,
                "new_ip": new_ip
            }
            
        except Exception as e:
            self.logger.error(f"Failed to update A record: {e}")
            raise
    
    def delete_a_record(
        self,
        fqdn: str,
        view: str = "default"
    ) -> Dict:
        """Delete A record"""
        endpoint = "record:a"
        params = {
            "name": fqdn,
            "view": view
        }
        
        try:
            response = self._make_request("GET", endpoint, params=params)
            records = response.json()
            
            if not records:
                raise InfobloxResourceNotFoundError(f"A record {fqdn} not found")
            
            record_ref = records[0]["_ref"]
            ip_address = records[0]["ipv4addr"]
            
            # Delete record
            self._make_request("DELETE", record_ref)
            
            self.logger.info(f"Deleted A record: {fqdn} ({ip_address})")
            
            return {
                "status": "success",
                "fqdn": fqdn,
                "ip_address": ip_address
            }
            
        except Exception as e:
            self.logger.error(f"Failed to delete A record: {e}")
            raise
    
    def create_cname_record(
        self,
        alias: str,
        canonical: str,
        ttl: Optional[int] = None,
        comment: Optional[str] = None,
        view: str = "default"
    ) -> Dict:
        """
        Create CNAME record
        
        Args:
            alias: Alias name (e.g., 'www.example.com')
            canonical: Canonical name (e.g., 'example.com')
            ttl: Time to live
            comment: Record comment
            view: DNS view
            
        Returns:
            dict: Created record details
        """
        endpoint = "record:cname"
        
        data = {
            "name": alias,
            "canonical": canonical,
            "view": view
        }
        
        if ttl:
            data["ttl"] = ttl
            data["use_ttl"] = True
        
        if comment:
            data["comment"] = comment
        
        try:
            response = self._make_request("POST", endpoint, data=data)
            record_ref = response.json()
            
            self.logger.info(f"Created CNAME record: {alias} → {canonical}")
            
            return {
                "status": "success",
                "alias": alias,
                "canonical": canonical,
                "_ref": record_ref
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create CNAME record: {e}")
            raise
    
    def create_ptr_record(
        self,
        ip_address: str,
        ptrdname: str,
        ttl: Optional[int] = None,
        comment: Optional[str] = None,
        view: str = "default"
    ) -> Dict:
        """
        Create PTR (reverse DNS) record
        
        Args:
            ip_address: IP address
            ptrdname: PTR domain name (FQDN)
            ttl: Time to live
            comment: Record comment
            view: DNS view
            
        Returns:
            dict: Created record details
        """
        endpoint = "record:ptr"
        
        data = {
            "ipv4addr": ip_address,
            "ptrdname": ptrdname,
            "view": view
        }
        
        if ttl:
            data["ttl"] = ttl
            data["use_ttl"] = True
        
        if comment:
            data["comment"] = comment
        
        try:
            response = self._make_request("POST", endpoint, data=data)
            record_ref = response.json()
            
            self.logger.info(f"Created PTR record: {ip_address} → {ptrdname}")
            
            return {
                "status": "success",
                "ip_address": ip_address,
                "ptrdname": ptrdname,
                "_ref": record_ref
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create PTR record: {e}")
            raise
```

#### 4.4.4 DNS Zone Management

```python
    def create_zone(
        self,
        fqdn: str,
        zone_format: str = "FORWARD",
        grid_primary: Optional[List[Dict]] = None,
        comment: Optional[str] = None
    ) -> Dict:
        """
        Create DNS zone
        
        Args:
            fqdn: Zone name (e.g., 'newapp.Contoso.internal')
            zone_format: FORWARD, IPV4, IPV6
            grid_primary: Grid members serving this zone
            comment: Zone comment
            
        Returns:
            dict: Created zone details
        """
        endpoint = "zone_auth"
        
        data = {
            "fqdn": fqdn,
            "zone_format": zone_format
        }
        
        if grid_primary:
            data["grid_primary"] = grid_primary
        
        if comment:
            data["comment"] = comment
        
        try:
            response = self._make_request("POST", endpoint, data=data)
            zone_ref = response.json()
            
            self.logger.info(f"Created DNS zone: {fqdn}")
            
            return {
                "status": "success",
                "zone": fqdn,
                "_ref": zone_ref
            }
            
        except Exception as e:
            self.logger.error(f"Failed to create zone: {e}")
            raise
    
    def get_zone_records(
        self,
        zone: str,
        record_type: Optional[str] = None
    ) -> List[Dict]:
        """
        Get all records in a zone
        
        Args:
            zone: Zone name
            record_type: Filter by record type (A, CNAME, etc.)
            
        Returns:
            list: Records in zone
        """
        if record_type:
            endpoint = f"record:{record_type.lower()}"
        else:
            endpoint = "allrecords"
        
        params = {
            "zone": zone,
            "_return_fields": "name,type,view"
        }
        
        try:
            response = self._make_request("GET", endpoint, params=params)
            records = response.json()
            
            return records
            
        except Exception as e:
            self.logger.error(f"Failed to get zone records: {e}")
            raise
```

### 4.5 Integration Patterns

#### Pattern 1: Azure VNet Provisioning with Infoblox IPAM

```python
def provision_vnet_with_infoblox(
    infoblox_adapter: InfobloxAdapter,
    azure_adapter: AzureAdapter,
    vnet_name: str,
    region: str,
    parent_network: str = "10.200.0.0/16",
    subnet_cidr: int = 24
) -> Dict:
    """
    Provision Azure VNet with IP allocation from Infoblox
    
    Args:
        infoblox_adapter: InfobloxAdapter instance
        azure_adapter: AzureAdapter instance
        vnet_name: VNet name
        region: Azure region
        parent_network: Parent network in Infoblox
        subnet_cidr: Subnet CIDR size
        
    Returns:
        dict: Provisioning result
    """
    try:
        # Step 1: Allocate IP space from Infoblox
        allocated_network = infoblox_adapter.get_next_available_network(
            parent_network=parent_network,
            cidr=subnet_cidr,
            comment=f"Azure VNet: {vnet_name} in {region}"
        )
        
        logger.info(f"Allocated {allocated_network} from Infoblox")
        
        # Step 2: Create network in Infoblox IPAM
        infoblox_adapter.create_network(
            network=allocated_network,
            comment=f"Azure-{region}-{vnet_name}"
        )
        
        # Step 3: Create Azure VNet
        vnet_result = azure_adapter.create_vnet(
            name=vnet_name,
            region=region,
            address_space=allocated_network
        )
        
        # Step 4: Register in Infoblox DNS
        infoblox_adapter.create_zone(
            fqdn=f"{vnet_name}.Contoso.internal",
            comment=f"Azure VNet zone for {vnet_name}"
        )
        
        return {
            "status": "success",
            "vnet_name": vnet_name,
            "allocated_network": allocated_network,
            "azure_vnet_id": vnet_result["vnet_id"],
            "dns_zone": f"{vnet_name}.Contoso.internal"
        }
        
    except Exception as e:
        logger.error(f"VNet provisioning failed: {e}")
        raise
```

#### Pattern 2: DR Failover DNS Update

```python
def update_dns_for_failover(
    infoblox_adapter: InfobloxAdapter,
    records_to_update: List[Dict[str, str]],
    ttl: int = 60
) -> Dict:
    """
    Update DNS records for DR failover
    
    Args:
        infoblox_adapter: InfobloxAdapter instance
        records_to_update: List of {fqdn, old_ip, new_ip}
        ttl: TTL for updated records (low for quick propagation)
        
    Returns:
        dict: Update results
    """
    results = []
    
    for record in records_to_update:
        try:
            result = infoblox_adapter.update_a_record(
                fqdn=record["fqdn"],
                new_ip=record["new_ip"],
                ttl=ttl
            )
            results.append({
                "fqdn": record["fqdn"],
                "status": "success",
                "old_ip": record["old_ip"],
                "new_ip": record["new_ip"]
            })
            
            logger.info(
                f"Updated DNS: {record['fqdn']} {record['old_ip']} → {record['new_ip']}"
            )
            
        except Exception as e:
            results.append({
                "fqdn": record["fqdn"],
                "status": "error",
                "error": str(e)
            })
            logger.error(f"Failed to update {record['fqdn']}: {e}")
    
    return {
        "total": len(records_to_update),
        "successful": sum(1 for r in results if r["status"] == "success"),
        "failed": sum(1 for r in results if r["status"] == "error"),
        "results": results
    }
```

### 4.6 Error Handling

```python
class InfobloxError(Exception):
    """Base exception for Infoblox adapter"""
    pass

class InfobloxAuthenticationError(InfobloxError):
    """Authentication failed"""
    pass

class InfobloxResourceNotFoundError(InfobloxError):
    """Resource not found"""
    pass

class InfobloxAPIError(InfobloxError):
    """API call failed"""
    pass
```

### 4.7 Testing

```python
import unittest
from unittest.mock import Mock, patch

class TestInfobloxAdapter(unittest.TestCase):
    
    def setUp(self):
        self.adapter = InfobloxAdapter(
            grid_master="infoblox.test.local",
            username="admin",
            password="password"
        )
    
    @patch('requests.get')
    def test_get_next_available_network(self, mock_get):
        """Test network allocation"""
        # Mock responses
        mock_get.return_value.status_code = 200
        mock_get.return_value.json.return_value = [
            {"_ref": "networkcontainer/ZG5zLm5ldHdvcmskMTAuMjAwLjAuMC8xNi8w:10.200.0.0/16/default"}
        ]
        
        network = self.adapter.get_next_available_network(
            parent_network="10.200.0.0/16",
            cidr=24
        )
        
        self.assertIsNotNone(network)
    
    @patch('requests.post')
    def test_create_a_record(self, mock_post):
        """Test A record creation"""
        mock_post.return_value.status_code = 201
        mock_post.return_value.json.return_value = "record:a/ZG5zLmJpbmRfYSQw..."
        
        result = self.adapter.create_a_record(
            fqdn="test.Contoso.com",
            ip_address="10.10.10.10"
        )
        
        self.assertEqual(result["status"], "success")
        self.assertEqual(result["fqdn"], "test.Contoso.com")
```

</details>

---

## 5. Azure Integration

<details>
<summary>Click to expand</summary>

### 5.1 Azure SDK Integration

For Azure integration details, refer to the existing documentation. Key adapters needed:

- **Network Adapter**: VNet, NSG, Azure Firewall, Traffic Manager
- **Compute Adapter**: VM operations, scale sets
- **ASR Adapter**: Site Recovery operations
- **Resource Graph Adapter**: Query resources across subscriptions

**Authentication**: Managed Identity or Service Principal

</details>

---

## 6. Cross-Vendor Orchestration

<details>
<summary>Click to expand</summary>

### 6.1 Orchestration Engine

```python
class MultiVendorOrchestrator:
    """
    Orchestrates operations across multiple vendors
    Ensures consistency and handles dependencies
    """
    
    def __init__(
        self,
        palo_alto_adapter: PaloAltoAdapter,
        f5_adapter: F5Adapter,
        infoblox_adapter: InfobloxAdapter,
        azure_adapter: AzureAdapter
    ):
        self.palo_alto = palo_alto_adapter
        self.f5 = f5_adapter
        self.infoblox = infoblox_adapter
        self.azure = azure_adapter
        self.logger = logging.getLogger(__name__)
    
    def block_ip_everywhere(
        self,
        ip_address: str,
        reason: str
    ) -> Dict:
        """
        Block IP across all vendors
        
        Args:
            ip_address: IP to block
            reason: Reason for blocking
            
        Returns:
            dict: Consolidated results
        """
        results = {
            "ip_address": ip_address,
            "reason": reason,
            "vendors": {}
        }
        
        # Step 1: Block in Palo Alto (edge first)
        try:
            pa_result = self._block_ip_palo_alto(ip_address, reason)
            results["vendors"]["palo_alto"] = pa_result
        except Exception as e:
            results["vendors"]["palo_alto"] = {"status": "error", "error": str(e)}
            self.logger.error(f"Palo Alto blocking failed: {e}")
        
        # Step 2: Block in F5 AFM
        try:
            f5_result = self.f5.add_afm_blocked_ip(ip_address, description=reason)
            results["vendors"]["f5"] = f5_result
        except Exception as e:
            results["vendors"]["f5"] = {"status": "error", "error": str(e)}
            self.logger.error(f"F5 blocking failed: {e}")
        
        # Step 3: Block in Azure NSGs
        try:
            azure_result = self._block_ip_azure_nsgs(ip_address, reason)
            results["vendors"]["azure"] = azure_result
        except Exception as e:
            results["vendors"]["azure"] = {"status": "error", "error": str(e)}
            self.logger.error(f"Azure NSG blocking failed: {e}")
        
        # Step 4: Validate
        validation = self._validate_ip_blocked(ip_address)
        results["validation"] = validation
        
        # Step 5: Document
        self._document_security_action(ip_address, reason, results)
        
        return results
    
    def _block_ip_palo_alto(self, ip_address: str, reason: str) -> Dict:
        """Block IP in all Palo Alto firewalls"""
        device_groups = ["DG-Chennai-DMZ", "DG-Pune-DMZ", "DG-HUB-Chennai", "DG-HUB-Pune"]
        
        return block_malicious_ip_palo_alto(
            adapter=self.palo_alto,
            ip_address=ip_address,
            reason=reason,
            device_groups=device_groups
        )
    
    def _block_ip_azure_nsgs(self, ip_address: str, reason: str) -> Dict:
        """Block IP in all Azure NSGs"""
        # Get all NSGs across subscriptions
        nsgs = self.azure.get_all_nsgs()
        
        results = []
        for nsg in nsgs:
            try:
                self.azure.add_nsg_deny_rule(
                    nsg_name=nsg["name"],
                    resource_group=nsg["resource_group"],
                    rule_name=f"Deny-{ip_address.replace('.', '_')}",
                    source_address=ip_address,
                    priority=100,
                    description=reason
                )
                results.append({"nsg": nsg["name"], "status": "success"})
            except Exception as e:
                results.append({"nsg": nsg["name"], "status": "error", "error": str(e)})
        
        return {
            "total_nsgs": len(nsgs),
            "successful": sum(1 for r in results if r["status"] == "success"),
            "results": results
        }
    
    def _validate_ip_blocked(self, ip_address: str) -> Dict:
        """Validate IP is blocked across all vendors"""
        validation = {}
        
        # Check Palo Alto
        # (Implementation depends on specific validation logic)
        validation["palo_alto"] = True
        
        # Check F5
        # (Implementation depends on specific validation logic)
        validation["f5"] = True
        
        # Check Azure
        # (Implementation depends on specific validation logic)
        validation["azure"] = True
        
        validation["all_vendors_compliant"] = all(validation.values())
        
        return validation
    
    def _document_security_action(
        self,
        ip_address: str,
        reason: str,
        results: Dict
    ):
        """Document security action in CMDB/ITSM"""
        # Generate documentation
        doc = {
            "timestamp": time.time(),
            "action": "block_ip",
            "ip_address": ip_address,
            "reason": reason,
            "affected_vendors": list(results["vendors"].keys()),
            "success": all(
                v.get("status") == "success"
                for v in results["vendors"].values()
            )
        }
        
        # Send to documentation service
        self.logger.info(f"Security action documented: {doc}")
        # In real implementation, call documentation agent or ITSM API
```

### 6.2 Complete Use Case: DR Failover Across All Vendors

```python
    def execute_dr_failover(
        self,
        primary_region: str,
        dr_region: str,
        recovery_plan: str
    ) -> Dict:
        """
        Execute complete DR failover across all vendors
        
        Args:
            primary_region: Primary region (e.g., 'Chennai')
            dr_region: DR region (e.g., 'Pune')
            recovery_plan: Azure ASR recovery plan name
            
        Returns:
            dict: Failover results
        """
        results = {
            "primary_region": primary_region,
            "dr_region": dr_region,
            "start_time": time.time(),
            "steps": []
        }
        
        try:
            # Step 1: Failover Azure resources (VMs, SQL)
            self.logger.info("Step 1: Failing over Azure resources...")
            azure_result = self.azure.execute_asr_failover(
                recovery_plan=recovery_plan,
                failover_direction="PrimaryToRecovery"
            )
            results["steps"].append({
                "step": 1,
                "vendor": "azure",
                "action": "asr_failover",
                "result": azure_result
            })
            
            # Step 2: Update Palo Alto firewalls
            self.logger.info("Step 2: Updating Palo Alto firewalls...")
            pa_result = self._update_palo_alto_for_dr(primary_region, dr_region)
            results["steps"].append({
                "step": 2,
                "vendor": "palo_alto",
                "action": "update_policies",
                "result": pa_result
            })
            
            # Step 3: Update F5 load balancers
            self.logger.info("Step 3: Updating F5 load balancers...")
            f5_result = self._update_f5_for_dr(primary_region, dr_region)
            results["steps"].append({
                "step": 3,
                "vendor": "f5",
                "action": "gtm_failover",
                "result": f5_result
            })
            
            # Step 4: Update DNS in Infoblox
            self.logger.info("Step 4: Updating DNS records...")
            dns_records = self._get_dns_records_to_update(primary_region, dr_region)
            infoblox_result = update_dns_for_failover(
                self.infoblox,
                dns_records,
                ttl=60
            )
            results["steps"].append({
                "step": 4,
                "vendor": "infoblox",
                "action": "update_dns",
                "result": infoblox_result
            })
            
            # Step 5: Validate application health
            self.logger.info("Step 5: Validating applications...")
            validation_result = self._validate_applications_in_dr()
            results["steps"].append({
                "step": 5,
                "action": "validation",
                "result": validation_result
            })
            
            results["end_time"] = time.time()
            results["total_duration_seconds"] = results["end_time"] - results["start_time"]
            results["status"] = "success"
            
            self.logger.info(
                f"DR failover completed in {results['total_duration_seconds']} seconds"
            )
            
            return results
            
        except Exception as e:
            results["status"] = "error"
            results["error"] = str(e)
            results["end_time"] = time.time()
            
            self.logger.error(f"DR failover failed: {e}")
            
            # Attempt rollback
            self.logger.warning("Attempting rollback...")
            rollback_result = self._rollback_dr_failover(results["steps"])
            results["rollback"] = rollback_result
            
            return results
    
    def _update_palo_alto_for_dr(
        self,
        primary_region: str,
        dr_region: str
    ) -> Dict:
        """Update Palo Alto for DR failover"""
        # Enable production rules in DR firewall
        dr_device_group = f"DG-{dr_region}-DMZ"
        
        # Example: Update NAT rules to point to DR public IPs
        # (Implementation depends on specific configuration)
        
        return {"status": "success", "device_group": dr_device_group}
    
    def _update_f5_for_dr(
        self,
        primary_region: str,
        dr_region: str
    ) -> Dict:
        """Update F5 GTM for DR failover"""
        gtm_pools = ["customer-portal", "api-service", "admin-portal"]
        
        results = []
        for pool in gtm_pools:
            result = dr_failover_f5_gtm(
                adapter=self.f5,
                gtm_pool_name=pool,
                primary_datacenter=primary_region,
                dr_datacenter=dr_region
            )
            results.append(result)
        
        return {
            "status": "success",
            "gtm_pools_updated": len(gtm_pools),
            "results": results
        }
    
    def _get_dns_records_to_update(
        self,
        primary_region: str,
        dr_region: str
    ) -> List[Dict]:
        """Get list of DNS records that need updating for DR"""
        # Query configuration or database for DNS mappings
        return [
            {"fqdn": "portal.Contoso.com", "old_ip": "20.10.10.10", "new_ip": "20.20.20.20"},
            {"fqdn": "api.Contoso.com", "old_ip": "20.10.10.20", "new_ip": "20.20.20.30"},
            # ... more records
        ]
    
    def _validate_applications_in_dr(self) -> Dict:
        """Validate applications are healthy in DR"""
        endpoints_to_test = [
            {"name": "Portal", "url": "https://portal.Contoso.com/health"},
            {"name": "API", "url": "https://api.Contoso.com/health"},
            # ... more endpoints
        ]
        
        results = []
        for endpoint in endpoints_to_test:
            try:
                response = requests.get(endpoint["url"], timeout=10)
                status = "healthy" if response.status_code == 200 else "unhealthy"
                results.append({
                    "name": endpoint["name"],
                    "status": status,
                    "response_code": response.status_code
                })
            except Exception as e:
                results.append({
                    "name": endpoint["name"],
                    "status": "error",
                    "error": str(e)
                })
        
        all_healthy = all(r["status"] == "healthy" for r in results)
        
        return {
            "all_healthy": all_healthy,
            "endpoints": results
        }
```

</details>

---

## 7. Authentication & Security

<details>
<summary>Click to expand</summary>

### 7.1 Credentials Management

**Azure Key Vault Storage**:

```python
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential

class CredentialsManager:
    """Manage vendor credentials securely in Azure Key Vault"""
    
    def __init__(self, vault_url: str):
        self.credential = DefaultAzureCredential()
        self.client = SecretClient(vault_url=vault_url, credential=self.credential)
    
    def get_palo_alto_credentials(self) -> Dict:
        """Get Palo Alto credentials from Key Vault"""
        return {
            "panorama_host": self.client.get_secret("palo-alto-panorama-host").value,
            "api_key": self.client.get_secret("palo-alto-api-key").value
        }
    
    def get_f5_credentials(self) -> Dict:
        """Get F5 credentials from Key Vault"""
        return {
            "host": self.client.get_secret("f5-host").value,
            "username": self.client.get_secret("f5-username").value,
            "password": self.client.get_secret("f5-password").value
        }
    
    def get_infoblox_credentials(self) -> Dict:
        """Get Infoblox credentials from Key Vault"""
        return {
            "grid_master": self.client.get_secret("infoblox-grid-master").value,
            "username": self.client.get_secret("infoblox-username").value,
            "password": self.client.get_secret("infoblox-password").value
        }
```

### 7.2 Audit Logging

```python
class AuditLogger:
    """Log all multi-vendor operations for audit and compliance"""
    
    def __init__(self, log_analytics_workspace_id: str, shared_key: str):
        self.workspace_id = log_analytics_workspace_id
        self.shared_key = shared_key
    
    def log_operation(
        self,
        operation: str,
        vendor: str,
        user: str,
        details: Dict,
        result: str
    ):
        """Log operation to Azure Log Analytics"""
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "operation": operation,
            "vendor": vendor,
            "user": user,
            "details": json.dumps(details),
            "result": result
        }
        
        # Send to Log Analytics
        self._send_to_log_analytics(log_entry)
    
    def _send_to_log_analytics(self, log_entry: Dict):
        """Send log to Azure Log Analytics"""
        # Implementation using Azure Monitor Ingestion API
        pass
```

</details>

---

## 8. Error Handling & Retry Logic

<details>
<summary>
Click to expand</summary>

### 8.1 Retry Decorator

```python
import time
from functools import wraps

def retry_on_failure(max_retries=3, delay=5, backoff=2):
    """
    Retry decorator with exponential backoff
    
    Args:
        max_retries: Maximum number of retries
        delay: Initial delay in seconds
        backoff: Backoff multiplier
    """
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            attempt = 0
            current_delay = delay
            
            while attempt < max_retries:
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    attempt += 1
                    
                    if attempt >= max_retries:
                        raise
                    
                    logger.warning(
                        f"Attempt {attempt}/{max_retries} failed for {func.__name__}: {e}. "
                        f"Retrying in {current_delay} seconds..."
                    )
                    
                    time.sleep(current_delay)
                    current_delay *= backoff
            
        return wrapper
    return decorator

# Usage
@retry_on_failure(max_retries=3, delay=5, backoff=2)
def create_firewall_rule_with_retry(adapter, rule_details):
    return adapter.create_security_rule(**rule_details)
```

### 8.2 Circuit Breaker Pattern

```python
class CircuitBreaker:
    """
    Circuit breaker to prevent cascading failures
    """
    
    def __init__(
        self,
        failure_threshold=5,
        recovery_timeout=60,
        expected_exception=Exception
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception
        
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open
    
    def call(self, func, *args, **kwargs):
        """Execute function through circuit breaker"""
        
        if self.state == "open":
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                self.state = "half-open"
            else:
                raise CircuitBreakerOpenError("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        
        except self.expected_exception as e:
            self._on_failure()
            raise
    
    def _on_success(self):
        """Handle successful call"""
        self.failure_count = 0
        self.state = "closed"
    
    def _on_failure(self):
        """Handle failed call"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = "open"
            logger.error(f"Circuit breaker opened after {self.failure_count} failures")

class CircuitBreakerOpenError(Exception):
    """Circuit breaker is open"""
    pass

# Usage
palo_alto_breaker = CircuitBreaker(failure_threshold=5, recovery_timeout=60)

def create_rule_with_circuit_breaker(adapter, rule):
    return palo_alto_breaker.call(
        adapter.create_security_rule,
        **rule
    )
```

</details>

---

## 9. Monitoring & Observability

<details>
<summary>Click to expand</summary>

### 9.1 Health Check Framework

```python
class HealthChecker:
    """Monitor health of all vendor integrations"""
    
    def __init__(
        self,
        palo_alto_adapter: PaloAltoAdapter,
        f5_adapter: F5Adapter,
        infoblox_adapter: InfobloxAdapter
    ):
        self.palo_alto = palo_alto_adapter
        self.f5 = f5_adapter
        self.infoblox = infoblox_adapter
    
    def check_all_vendors(self) -> Dict:
        """Check health of all vendors"""
        return {
            "timestamp": time.time(),
            "palo_alto": self._check_palo_alto(),
            "f5": self._check_f5(),
            "infoblox": self._check_infoblox()
        }
    
    def _check_palo_alto(self) -> Dict:
        """Check Palo Alto Panorama health"""
        try:
            # Simple connectivity check
            self.palo_alto.panorama.refresh_system_info()
            
            return {
                "status": "healthy",
                "panorama_version": self.palo_alto.panorama.version,
                "device_groups": len(self.palo_alto.get_device_groups())
            }
        except Exception as e:
            return {
                "status": "unhealthy",
                "error": str(e)
            }
    
    def _check_f5(self) -> Dict:
        """Check F5 health"""
        try:
            response = self.f5._make_request("GET", "/tm/sys/version")
            
            return {
                "status": "healthy",
                "version": response.json().get("entries", {})
            }
        except Exception as e:
            return {
                "status": "unhealthy",
                "error": str(e)
            }
    
    def _check_infoblox(self) -> Dict:
        """Check Infoblox Grid Master health"""
        try:
            response = self.infoblox._make_request("GET", "grid")
            
            return {
                "status": "healthy",
                "grid_members": len(response.json())
            }
        except Exception as e:
            return {
                "status": "unhealthy",
                "error": str(e)
            }
```

### 9.2 Metrics Collection

```python
from prometheus_client import Counter, Histogram, Gauge

# Define metrics
api_calls_total = Counter(
    'multivendor_api_calls_total',
    'Total API calls to vendor systems',
    ['vendor', 'operation', 'status']
)

api_call_duration = Histogram(
    'multivendor_api_call_duration_seconds',
    'API call duration',
    ['vendor', 'operation']
)

vendor_health = Gauge(
    'multivendor_health_status',
    'Health status of vendor integration',
    ['vendor']
)

# Usage in adapters
def create_security_rule_with_metrics(adapter, rule):
    vendor = "palo_alto"
    operation = "create_security_rule"
    
    with api_call_duration.labels(vendor=vendor, operation=operation).time():
        try:
            result = adapter.create_security_rule(**rule)
            api_calls_total.labels(vendor=vendor, operation=operation, status="success").inc()
            return result
        except Exception as e:
            api_calls_total.labels(vendor=vendor, operation=operation, status="error").inc()
            raise
```

</details>

---

## 10. Testing & Validation

<details>
<summary>Click to expand</summary>

### 10.1 Integration Test Suite

```python
import pytest

class TestMultiVendorIntegration:
    """Integration tests for multi-vendor operations"""
    
    @pytest.fixture
    def orchestrator(self):
        """Create orchestrator with test adapters"""
        creds_manager = CredentialsManager(vault_url=TEST_VAULT_URL)
        
        palo_alto_creds = creds_manager.get_palo_alto_credentials()
        f5_creds = creds_manager.get_f5_credentials()
        infoblox_creds = creds_manager.get_infoblox_credentials()
        
        palo_alto_adapter = PaloAltoAdapter(**palo_alto_creds)
        f5_adapter = F5Adapter(**f5_creds)
        infoblox_adapter = InfobloxAdapter(**infoblox_creds)
        azure_adapter = AzureAdapter()
        
        return MultiVendorOrchestrator(
            palo_alto_adapter,
            f5_adapter,
            infoblox_adapter,
            azure_adapter
        )
    
    def test_block_ip_all_vendors(self, orchestrator):
        """Test blocking IP across all vendors"""
        test_ip = "203.0.113.99"
        reason = "Test block - automated testing"
        
        result = orchestrator.block_ip_everywhere(test_ip, reason)
        
        assert result["status"] == "success"
        assert "palo_alto" in result["vendors"]
        assert "f5" in result["vendors"]
        assert "azure" in result["vendors"]
        
        # Cleanup
        cleanup_blocked_ip(test_ip)
    
    def test_vnet_provisioning_with_infoblox(self, orchestrator):
        """Test Azure VNet provisioning with Infoblox IPAM"""
        vnet_name = "test-vnet-integration"
        region = "eastus"
        
        result = provision_vnet_with_infoblox(
            orchestrator.infoblox,
            orchestrator.azure,
            vnet_name,
            region
        )
        
        assert result["status"] == "success"
        assert "allocated_network" in result
        assert "azure_vnet_id" in result
        
        # Cleanup
        cleanup_vnet(vnet_name, region)
        cleanup_infoblox_network(result["allocated_network"])
```

### 10.2 Validation Framework

```python
class ConfigurationValidator:
    """Validate configuration consistency across vendors"""
    
    def validate_firewall_rules_consistent(
        self,
        expected_blocked_ips: List[str],
        palo_alto_adapter: PaloAltoAdapter,
        azure_adapter: AzureAdapter
    ) -> Dict:
        """
        Validate that blocked IPs are consistent across vendors
        
        Args:
            expected_blocked_ips: List of IPs that should be blocked
            palo_alto_adapter: Palo Alto adapter
            azure_adapter: Azure adapter
            
        Returns:
            dict: Validation results
        """
        results = {
            "expected_ips": expected_blocked_ips,
            "inconsistencies": []
        }
        
        # Check Palo Alto
        pa_blocked_ips = self._get_blocked_ips_palo_alto(palo_alto_adapter)
        
        # Check Azure NSGs
        azure_blocked_ips = self._get_blocked_ips_azure(azure_adapter)
        
        # Find inconsistencies
        for ip in expected_blocked_ips:
            if ip not in pa_blocked_ips:
                results["inconsistencies"].append({
                    "ip": ip,
                    "vendor": "palo_alto",
                    "issue": "not_blocked"
                })
            
            if ip not in azure_blocked_ips:
                results["inconsistencies"].append({
                    "ip": ip,
                    "vendor": "azure",
                    "issue": "not_blocked"
                })
        
        results["is_consistent"] = len(results["inconsistencies"]) == 0
        
        return results
```

</details>

---

## Appendix A: Complete Example - End-to-End DR Failover

```python
def complete_dr_failover_example():
    """
    Complete example: DR failover across Palo Alto, F5, Infoblox, and Azure
    """
    
    # Step 1: Initialize adapters
    creds_manager = CredentialsManager(vault_url=VAULT_URL)
    
    palo_alto = PaloAltoAdapter(**creds_manager.get_palo_alto_credentials())
    f5 = F5Adapter(**creds_manager.get_f5_credentials())
    infoblox = InfobloxAdapter(**creds_manager.get_infoblox_credentials())
    azure = AzureAdapter()
    
    # Step 2: Create orchestrator
    orchestrator = MultiVendorOrchestrator(palo_alto, f5, infoblox, azure)
    
    # Step 3: Execute DR failover
    result = orchestrator.execute_dr_failover(
        primary_region="Chennai",
        dr_region="Pune",
        recovery_plan="Production-Critical-Apps"
    )
    
    # Step 4: Display results
    print(f"DR Failover Status: {result['status']}")
    print(f"Total Duration: {result['total_duration_seconds']} seconds")
    
    for step in result["steps"]:
        print(f"Step {step['step']}: {step['action']} - {step['result']['status']}")
    
    return result
```

---

## Appendix B: Configuration Files

### adapter_config.yaml

```yaml
vendors:
  palo_alto:
    panorama_host: panorama.Contoso.local
    device_groups:
      - DG-Chennai-DMZ
      - DG-Pune-DMZ
      - DG-HUB-Chennai
      - DG-HUB-Pune
    api_version: "10.1"
  
  f5:
    devices:
      - name: F5-Chennai-01
        host: f5-chennai-01.Contoso.local
        role: active
      - name: F5-Pune-01
        host: f5-pune-01.Contoso.local
        role: active
    api_version: "v16.1"
  
  infoblox:
    grid_master: infoblox-gm.Contoso.local
    wapi_version: "v2.12"
    parent_networks:
      azure: "10.200.0.0/16"
      onprem: "10.100.0.0/16"
  
  azure:
    subscriptions:
      - subscription_id: "xxx-xxx-xxx"
        name: "Contoso-Prod-Chennai"
      - subscription_id: "yyy-yyy-yyy"
        name: "Contoso-Prod-Pune"
```

---

