# Azure Cross-Region LB Architecture Lab Setup Guide

##  Goal
Build a working environment matching the Azure CR-LB architecture exactly as documented:
- 2 regions (USA + India)
- 4 Brokers (B1, B2, B3, B4)
- 4 Standard LBs (one per Broker)
- 1 Cross-Region LB (with 4 frontends)
- DNS entries mapping each broker to its CR-LB frontend
- Basic TCP testing (mTLS optional)

---

##  1. Create Resource Groups
Create 3 resource groups:

- **rg-us-brokers** — USA region resources  
- **rg-in-brokers** — India region resources  
- **rg-global-brokers** — CR-LB + DNS zone  

---

##  2. Create VNets

### USA Region
- **vnet-us-brokers** (10.10.0.0/16)
  - Subnets:
    - **broker-b1-us-snet**
    - **broker-b2-us-snet**

### India Region
- **vnet-in-brokers** (10.20.0.0/16)
  - Subnets:
    - **broker-b3-in-snet**
    - **broker-b4-in-snet**

---

##  3. Create Broker VMs
Create **4 Ubuntu VMs**, each acting as a test Broker:

- **broker-b1-us**
- **broker-b2-us**
- **broker-b3-in**
- **broker-b4-in**

Requirements:
- Private IP only  
- Each VM placed in its region-specific subnet  
- NSG rule: Allow **TCP 443 inbound** only from its corresponding SLB  

Optional:
- Install a simple TCP listener (e.g., Socat or Ncat) to simulate Broker behavior.

---

##  4. Create Standard Load Balancers (1 per Broker)
Each Broker needs its **own** Standard Public Load Balancer.

### USA Region
- **slb-b1-us** → backend: broker-b1-us  
- **slb-b2-us** → backend: broker-b2-us  

### India Region
- **slb-b3-in** → backend: broker-b3-in  
- **slb-b4-in** → backend: broker-b4-in  

Each SLB must have:
- **SKU:** Standard  
- **Health Probe:** TCP 443  
- **LB Rule:** 443 → 443  

Backend pool should contain **only one VM** (its dedicated Broker).

---

##  5. Collect SLB Public IPs
Record the 4 public IPs assigned to:

- slb-b1-us  
- slb-b2-us  
- slb-b3-in  
- slb-b4-in  

**These will be used as backend endpoints for the Cross-Region Load Balancer.**

---

##  6. Create the Cross-Region Load Balancer
In **rg-global-brokers**, create:

### **Cross-Region Load Balancer**
- Name: **crlb-brokers**
- SKU: **Standard**
- Type: **Public (Global)**

### Create 4 Frontend IP Configurations
Each frontend represents a Broker hostname:

- **fe-b1-us**
- **fe-b2-us**
- **fe-b3-in**
- **fe-b4-in**

Each frontend gets its own **Anycast IP**.

### Create 4 Backend Pools
Each maps to a specific regional SLB:

- **be-b1-us** → slb-b1-us public IP  
- **be-b2-us** → slb-b2-us public IP  
- **be-b3-in** → slb-b3-in public IP  
- **be-b4-in** → slb-b4-in public IP  

### Create 4 Load-Balancing Rules
Each rule connects:

- One CR-LB frontend  
- One backend pool  
- Port 443 over TCP  

Example:
- **rule-b1-us:** fe-b1-us → be-b1-us → TCP 443  

Repeat for b2, b3, b4.

## Detailed info for CR-LB Creatio
### Step-by-Step Guide: Creating an Azure Cross-Region Load Balancer (CR-LB)

This segment explains how to build a **fully working Cross-Region Load Balancer** for the RC system architecture.  

---

# Prerequisites
Before you begin, you must already have:

### **Resource Groups**
- `rg-rc-global` (for CR-LB + DNS)
- `rg-us-brokers`
- `rg-in-brokers`

### **Standard Public Load Balancers (one per Broker)**
Already created in each region:
- US: `slb-b1-us`, `slb-b2-us`
- India: `slb-b3-in`, `slb-b4-in`

Each SLB must:
- Be **Standard SKU**  
- Be **Public**  
- Have **1 backend VM** (the Broker)  
- Have a **TCP health probe** (e.g., port 443)  
- Have a **TCP LB rule** (443 → 443)

---

# Part A — Create the Cross-Region Load Balancer

1. In Azure Portal search for **Load balancer** → **Create**.
2. Fill out:
   - **Subscription:** your subscription  
   - **Resource group:** `rg-rc-global`  
   - **Name:** `crlb-brokers`  
   - **Region:** (automatically shows *Global*)  
   - **SKU:** Standard  
   - **Type:** Public  
3. Click **Review + Create** → **Create**.

You now have an **empty global load balancer**.

---

#  Part B — Create CR-LB Frontend IPs (One Per Broker)

You will create **four Anycast frontends**:

| Frontend Name | Represents | Broker |
|---------------|------------|--------|
| fe-b1-us | US Broker 1 | B1 |
| fe-b2-us | US Broker 2 | B2 |
| fe-b3-in | India Broker 3 | B3 |
| fe-b4-in | India Broker 4 | B4 |

### To create each frontend:

1. Open **crlb-brokers**
2. Go to **Frontend IP configuration**
3. Click **+ Add**
4. Fill in:
   - **Name:** `fe-b1-us` (or b2-us, b3-in, b4-in)
   - **IP Version:** IPv4  
   - **Public IP Address:** **Create new**
     - Name: `pip-fe-b1-us`
     - **SKU: Standard**
     - **Tier: Global**
     - **Assignment: Static**
   - Click **OK**
5. Click **Add**

Repeat for all four frontends.

Each frontend gets its own **Anycast IP address**.

---

# Part C — Create Backend Pools (One Per Broker)

Each backend pool maps the CR-LB frontend → **regional SLB** → Broker.

You will create:

| Backend pool | Points to SLB |
|--------------|----------------|
| be-b1-us | slb-b1-us |
| be-b2-us | slb-b2-us |
| be-b3-in | slb-b3-in |
| be-b4-in | slb-b4-in |

### Steps:

1. Open **crlb-brokers**
2. Go to **Backend pools** → **+ Add**
3. Enter:
   - **Name:** `be-b1-us`
   - **Backend type:** Regional load balancers
4. Under **Add load balancer**:
   - **Subscription:** same
   - **Region:** Select USA region  
   - **Load balancer:** choose `slb-b1-us`
   - **Frontend:** choose the Public Frontend of `slb-b1-us`
5. Click **Add**

Repeat for b2-us, b3-in, b4-in.

---

# Part D — Create Load-Balancing Rules (Front-end ↔ Backend pair)

Each rule binds:
- One CR-LB frontend IP
- One backend pool
- One TCP port (443)

You will create **4 rules**:

| Rule name | Frontend | Backend | Port |
|-----------|----------|----------|------|
| rule-b1-us | fe-b1-us | be-b1-us | 443 |
| rule-b2-us | fe-b2-us | be-b2-us | 443 |
| rule-b3-in | fe-b3-in | be-b3-in | 443 |
| rule-b4-in | fe-b4-in | be-b4-in | 443 |

### Steps:
1. Open **crlb-brokers**
2. Go to **Load balancing rules** → **+ Add**
3. Fill in:
   - **Name:** `rule-b1-us`
   - **Frontend IP:** `fe-b1-us`
   - **Backend pool:** `be-b1-us`
   - **Protocol:** TCP
   - **Port:** 443
4. Click **Add**

Repeat for all brokers.

---

# Part E — (Optional) Failover to a Secondary Region

If required:
- Open a backend pool
- Add a **secondary SLB** to the pool
- Enable **Priority** mode

This allows a broker hostname to fall back to a different region.

---

# Part F — DNS Records (Broker FQDN → CR-LB Frontends)

Create these A records (Azure DNS or external DNS):

| Hostname | Points to |
|----------|-----------|
| broker-us-1.company.com | Public IP of `fe-b1-us` |
| broker-us-2.company.com | Public IP of `fe-b2-us` |
| broker-in-3.company.com | Public IP of `fe-b3-in` |
| broker-in-4.company.com | Public IP of `fe-b4-in` |

---

# Part G — Validate End-to-End

From any external client:

### 1. DNS resolution
```bash
nslookup broker-us-1.company.com
```

Should return the **Anycast IP** of `fe-b1-us`.

### 2. TLS connect test
```bash
openssl s_client -connect broker-us-1.company.com:443
```

### 3. HTTP/TCP test
```bash
curl -v https://broker-us-1.company.com
```

Traffic should flow:

Client → nearest Azure Edge (Anycast) → CR-LB → regional SLB → Broker VM

---

# Common Mistakes (Read Before Troubleshooting)

### ❌ Using Basic SKU Load Balancer  
CR-LB supports **Standard SKU only**.

### ❌ Putting both Brokers in one SLB  
Each Broker **must** have its own SLB.

### ❌ Forgetting the SLB’s health probe  
CR-LB refuses to send traffic if the SLB is unhealthy.

### ❌ Trying to attach VMs directly to CR-LB  
CR-LB → **regional SLBs only**.

# You now have a complete CR-LB setup ready for RC system testing.


---

##  7. DNS Entries
In Azure DNS (rg-global-brokers), create **A** records:

| Hostname | Points To |
|----------|-----------|
| broker-us-1.company.com | fe-b1-us Anycast IP |
| broker-us-2.company.com | fe-b2-us Anycast IP |
| broker-in-3.company.com | fe-b3-in Anycast IP |
| broker-in-4.company.com | fe-b4-in Anycast IP |

These DNS records allow Targets and Controllers to reach the correct CR-LB frontend.

---

##  8. Testing Connectivity
From any client machine:

### DNS resolution
```
nslookup broker-us-1.company.com
```

### TLS connectivity test
```
openssl s_client -connect broker-us-1.company.com:443
```

### Basic HTTPS/TCP test
```
curl -v https://broker-us-1.company.com
```

Expected:
- Traffic enters nearest Azure POP  
- Routed via CR-LB → correct region → SLB → Broker VM  

---

##  9. Optional Enhancements
- Enable mTLS authentication  
- Integrate Azure Key Vault for cert rotation  
- Enable Network Watcher + NSG Flow Logs  
- Add Azure Monitor alerts  
- Replace VMs with **VMSS** or **AKS** for auto-scaling  

---

**End of Lab Setup Guide**
