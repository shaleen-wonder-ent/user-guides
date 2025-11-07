# Azure Cross-Region Load Balancer Architecture for RC System

## Architecture Overview

### DNS Configuration
- Each broker has a unique hostname with Anycast IP:
  - `broker-us-1.company.com` → CR-LB Frontend A (Anycast)
  - `broker-us-2.company.com` → CR-LB Frontend B (Anycast)
  - `broker-in-3.company.com` → CR-LB Frontend C (Anycast)
  - `broker-in-4.company.com` → CR-LB Frontend D (Anycast)

### Traffic Flow Components

**1. Azure Cross-Region Load Balancer (CR-LB)**
- Global Anycast IP per broker hostname
- BGP-advertised from multiple Azure edge locations
- Routes traffic to appropriate regional backend

**2. Regional Standard Load Balancers**
- **US Region**: 
  - Standard LB → Backend pool [B1, B2]
- **India Region**: 
  - Standard LB → Backend pool [B3, B4]

**3. Azure Traffic Manager** (Optional)
- Health monitoring for broker availability
- Automatic failover if broker becomes unhealthy

---

## Connection Flow: USA Controller (C1) → India Target (T1)

### Step 1: Initial Connection Code Request
**C1 (USA) → B1 (USA)**

1. C1 resolves `broker-us-1.company.com` → CR-LB Anycast Frontend A
2. Traffic enters Azure at nearest USA POP
3. CR-LB routes to USA Regional Standard LB
4. Standard LB forwards to B1
5. B1 generates connection code, returns to C1

### Step 2: Target Latency Testing
**T1 (India) tests all 4 brokers**

1. T1 initiates TCP connections to all 4 hostnames:
   - `broker-us-1.company.com`
   - `broker-us-2.company.com`
   - `broker-in-3.company.com`
   - `broker-in-4.company.com`

2. Each resolves to Anycast IP → enters Azure at nearest India POP

3. T1 measures end-to-end latency:
   - `broker-us-*` → India POP → Azure backbone → USA → ~150-200ms
   - `broker-in-*` → India POP → Azure backbone → India → ~10-20ms

4. **T1 selects B3** (`broker-in-3.company.com`) - lowest latency

### Step 3: Target Establishes Connection
**T1 (India) → B3 (India)**

1. T1 connects to `broker-in-3.company.com` with connection code
2. Traffic enters Azure at India POP
3. CR-LB routes to India Regional Standard LB
4. Standard LB forwards to B3
5. B3 validates connection code and notifies broker mesh

### Step 4: Broker Mesh Coordination
**B1 ↔ B3 (Broker-to-Broker communication)**

1. B3 notifies broker mesh: "Target T1 connected with code XYZ"
2. B1 (holding C1's connection) receives notification
3. B1 instructs C1: "Reconnect to `broker-in-3.company.com`"

### Step 5: Controller Reconnection
**C1 (USA) → B3 (India)**

1. C1 resolves `broker-in-3.company.com` → CR-LB Anycast Frontend C
2. Traffic enters Azure at nearest USA POP
3. CR-LB routes internally via Azure backbone to India
4. India Regional Standard LB forwards to B3
5. B3 establishes mTLS connection with C1

### Step 6: Active Remote Session
**C1 (USA) ↔ B3 (India) ↔ T1 (India)**

- Two active TCP+mTLS connections:
  - `C1 → B3`: USA POP → Azure backbone → India (optimized path)
  - `T1 → B3`: India POP → local India region (minimal latency)
- B3 relays remote control protocol between C1 and T1
- Lowest possible latency achieved via Azure's global network

---

## Key Benefits

✅ **Anycast Entry**: All clients enter Azure at nearest POP
✅ **Azure Backbone**: Internal routing faster than public internet  
✅ **Accurate Selection**: Latency testing reflects real traffic path
✅ **Regional Optimization**: Targets connect to nearest broker
✅ **L4 Pass-through**: No TLS termination at LB, end-to-end encryption
✅ **Horizontal Scaling**: Easy to add brokers in new regions
✅ **Fault Tolerance**: Multiple brokers per region + health monitoring

---

## Future Expansion

To reduce latency for other regions, deploy additional broker pairs:

- **EMEA**: `broker-eu-5.company.com`, `broker-eu-6.company.com` (West Europe)
- **LATAM**: `broker-br-7.company.com`, `broker-br-8.company.com` (Brazil South)
- **APAC**: `broker-sg-9.company.com`, `broker-sg-10.company.com` (Singapore)

Each follows the same CR-LB + Regional Standard LB pattern.

