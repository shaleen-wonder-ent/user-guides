# Azure Traffic Manager (ATM) Architecture for RC System – End-to-End Communication Flow

## Overview

This document describes how to architect the RC (Remote Control) system using **Azure Traffic Manager (ATM)** instead of a Cross-Region Load Balancer (CR-LB). The architecture shows how the Controller and Target connect to Brokers in different regions, with ATM serving as the global DNS-based traffic distributor. It also explains the key trade-offs compared to a CR-LB-based architecture.

---

## 1. Architecture Diagram

<details>
  <summary> High Level(Simple)</summary>
  
<img width="3188" height="747" alt="image" src="https://github.com/user-attachments/assets/53d25a48-cd11-42a4-9b98-2ad6e16810eb" />


</details>

---

<details>
  <summary> Flow Level</summary>
  
<img width="1092" height="1005" alt="image" src="https://github.com/user-attachments/assets/14a23968-b970-4afd-941c-1648f4d878cc" />


</details>

---

## 2. Key Components

- **RC Brokers**: Broker VMs deployed in multiple Azure Regions (e.g., US, India). Each is fronted by a Regional Standard Load Balancer (LB) with a Regional Public IP.
- **Azure Standard Load Balancer (Regional)**: Provides L4 TCP forwarding to brokers in each region.
- **Azure Traffic Manager (ATM)**: DNS-based global load balancer. Routes DNS queries to the best available regional broker endpoint, using methods like Performance, Priority, or Weighted.
- **RC Controller**: Support engineer's client (e.g., in USA), initiates sessions.
- **RC Target**: End-user's system (e.g., in India), targeted for remote session.
- **RC Server**: Used for start/informational data, not involved in session traffic.

---

## How this architecture works? (flow, simplified)

## What is Azure Traffic Manager?

**Azure Traffic Manager (ATM)** is like a **smart phone directory** for our broker servers. When a user (Target or Controller) wants to connect to a broker, they ask ATM, *"Which broker should I connect to?"* ATM looks at where the user is located, checks which brokers are healthy, and gives them the best broker's address.

---

## Real-World Example: Connecting to Broker B3

Let's walk through how **Target (in India)** connects to **Broker B3** using Azure Traffic Manager.

---

### **Step 1: Target Wants to Connect**
- **Target (in India)** needs to connect to a broker for a remote session.
- Target has 4 broker options:
  - `broker-in-3.company.com` (India)
  - `broker-in-4.company.com` (India)
  - `broker-us-1.company.com` (USA)
  - `broker-us-2.company.com` (USA)

---

### **Step 2: Target Asks ATM for the Broker's Address**
- Target asks: *"What's the address for broker-in-3.company.com?"*
- This question goes to **Azure Traffic Manager** (not a regular DNS).

**Think of it like:**
- Target calls the phone directory (ATM) and asks for the broker's phone number.

---

### **Step 3: ATM Gives Target the Best Broker Address**
- ATM checks:
  -  Is `broker-in-3.company.com` healthy?
  -  Where is Target located? (India)
  -  Which server is closest and fastest? (India Load Balancer)

- ATM responds: *"broker-in-3.company.com is at IP address 20.192.45.100"*

**Think of it like:**
- The directory says, *"Call this number: 20.192.45.100"*

---

### **Step 4: Target Connects Directly to the Broker**
- Target now has the address: **20.192.45.100**
- Target makes a **direct connection** to that address.
- **Important:** ATM is done at this point—it's NOT involved in the actual conversation.

**Think of it like:**
- After getting the phone number, you call directly. The directory is not on the call.

---

### **Step 5: Load Balancer Passes the Call to Broker B3**
- The address **20.192.45.100** belongs to a **Load Balancer** in India.
- The Load Balancer forwards Target's connection to **Broker B3**.

**Think of it like:**
- You call a company's main number, and the receptionist transfers you to the right person (Broker B3).

---

### **Step 6: Target is Connected to Broker B3**
- Target (India) is now talking to **Broker B3**.
- They can now start the remote session.

---

## The Same Process for Controller (USA)

**Controller (USA)** wants to connect to the same **Broker B3**:

1. **Controller asks ATM:** *"What's the address for broker-in-3.company.com?"*
2. **ATM responds:** *"It's at 20.192.45.100"* (same India Load Balancer)
3. **Controller connects directly** to 20.192.45.100 from USA over the internet
4. **Load Balancer** forwards to **Broker B3**
5. **Controller is now connected** to Broker B3

Now both **Target (India)** and **Controller (USA)** are connected to the same **Broker B3**, and they can have their remote session.

---

## Key Points (Simplified)

| What | Explanation |
|------|-------------|
| **ATM's Job** | Acts like a phone directory—tells you which broker to connect to |
| **How it works** | You ask for a broker's name, ATM gives you its IP address |
| **ATM is NOT in the call** | After giving you the address, ATM steps away—you connect directly |
| **Smart routing** | ATM picks the best broker based on your location and broker health |
| **DNS-based** | Works using DNS (like how websites work—you type a name, get an IP) |

---

## Why Do We Use ATM?

### **Benefits:**
 **Automatic routing**: Users get connected to the nearest healthy broker  
 **No special IPs needed**: Works with regular Azure Load Balancers  
 **Health checks**: Automatically avoids broken brokers  
 **Global coverage**: Works anywhere in the world  

### **Limitations:**
 **Not the fastest option**: Traffic goes over public internet (not Azure's private network)  
 **Slower failover**: If a broker fails, it takes time for users to switch (DNS caching)  
 **Variable performance**: Users might not always connect to the absolute closest broker  

---

## When Should We Use ATM?

 **Use ATM when:**
- We can't get special Global IP addresses
- We need global routing with health checks
- DNS-based routing is acceptable

 **Use Cross-Region Load Balancer (better option) when:**
- We have Global IP addresses available
- We need the fastest possible connections
- We need instant failover

---

**Azure Traffic Manager** is like a smart directory that helps users find and connect to the best broker server. It works by:

1. **User asks:** "Which broker should I use?"
2. **ATM answers:** "Use this IP address (the best one for you)"
3. **User connects directly** to that broker
4. **Session established** between Target and Controller

**ATM only handles the "finding" part—it doesn't touch the actual connection.**

---
  

## 3. End-to-End Communication Flow (bit more depth)

### **A. Setup**

1. **Each Broker DNS name (e.g., broker-us-1.company.com)** is configured as an endpoint in Azure Traffic Manager.
2. **Azure Traffic Manager** profile contains all broker endpoints pointing to Regional Standard LB Public IPs:
   - `broker-in-3.company.com` → ATM endpoint → `slb-b3-in` Public IP
   - `broker-in-4.company.com` → ATM endpoint → `slb-b4-in` Public IP
   - `broker-us-1.company.com` → ATM endpoint → `slb-b1-us` Public IP
   - `broker-us-2.company.com` → ATM endpoint → `slb-b2-us` Public IP
3. **Routing Method**: 'Performance' or 'Priority'; **'Performance'** recommended.
4. **Health probing** (TCP/443) checks endpoint status.

---

### **B. Controller/Target Connectivity**

**Step 1: Controller (C1, USA) needs to initiate a remote session.**

- C1 queries DNS for the broker hostname (e.g., `broker-us-1.company.com`).
- **Azure Traffic Manager** receives the DNS query and responds with the **Public IP of the best regional Standard Load Balancer** based on:
  - Performance (latency from DNS resolver)
  - Health probe status
  - Configured routing policy
- Controller receives the IP address (e.g., `20.x.x.x` for `slb-b1-us`).
- Controller establishes a **direct TCP+TLS connection** to the regional Standard LB's public IP.
- **Important**: ATM is **NOT in the data path**—it only provides DNS resolution.
- The connection enters Azure in the region of the resolved broker (not always at the nearest edge).

**Step 2: Target (T1, India) prepares for remote session.**

- T1 queries DNS for all broker hostnames via ATM.
- ATM returns the best regional broker endpoint IP for each query (e.g., India Broker IPs for T1).
- T1 tests all 4 brokers (or a subset), selects the one with the best network performance.
- T1 establishes a **direct TCP+TLS connection** to the selected broker's regional LB public IP.

**Step 3: Brokers coordinate via mesh and connect Controller and Target as needed.**

- Controller (USA) and Target (India) may be connected to the same or different brokers, depending on ATM routing and network latency.
- Broker mesh handles session handover if needed.

---

## 4. Traffic Flow Clarification

### **Important: ATM is DNS-Only, Not in Data Path**

**Step-by-step flow:**

```
1. Target/Controller → DNS Query for "broker-us-1.company.com" → Azure Traffic Manager
2. Azure Traffic Manager → DNS Response: "slb-b1-us Public IP = 20.x.x.x" → Target/Controller
3. Target/Controller → TCP+TLS connection directly to 20.x.x.x (slb-b1-us) → slb-b1-us → Broker B1
```

**Key Points:**
- **ATM is only involved in DNS resolution (Steps 1-2)**, not in the actual data connection.
- After receiving the IP from ATM, the client connects **directly to the regional Standard Load Balancer**.
- Traffic does **not** flow through ATM—it goes straight from client to regional LB to broker.

### **Flow Summary Table**

| Step | Action | Goes Through ATM? |
|------|--------|-------------------|
| 1. DNS Query | Client queries broker hostname | ✅ Yes (DNS only) |
| 2. DNS Response | ATM returns regional SLB IP | ✅ Yes (DNS only) |
| 3. TCP Connection | Client connects to SLB IP | ❌ No (direct to SLB) |
| 4. Traffic Flow | SLB forwards to Broker VM | ❌ No (regional only) |

---

## 5. Sequence Diagram of Traffic Manager-based Routing

```
Client (C1) or Target (T1)
    |
    |---(DNS query for broker-us-1.company.com)---> Azure Traffic Manager (ATM)
    |<--(ATM returns slb-b1-us public IP: 20.x.x.x)---
    |
    |---(TCP+TLS connect directly to 20.x.x.x)---> Regional Standard LB (slb-b1-us)
    |                                                      |
    |                                                      v
    |                                                 Broker B1 VM
    |
    |<---------[Remote Session Established via Broker Mesh]-----------------
```

---

## 6. Trade-Offs: Azure Traffic Manager vs. Cross-Region Load Balancer (CR-LB)

| Feature                         | CR-LB (Ideal)                                            | ATM (Fallback/Current)                                        |
|----------------------------------|---------------------------------------------------------|---------------------------------------------------------------|
| Entry Point to Azure             | Nearest Azure Edge POP (Anycast IP)                     | Traffic enters at broker's home region (regional public IP)   |
| Global Single IP                 | ✅ (Anycast/Global IP per Broker)                        | ❌ (Returns regional IP per DNS response)                     |
| Routing Control                  | L4, in-path; instant failover, Azure backbone routing   | DNS-based; depends on DNS cache, latency to DNS resolver      |
| Protocol Support                 | TCP, mTLS, any L4                                       | TCP, mTLS, any protocol (since ATM is DNS-only)              |
| Optimal Latency                  | Yes (always nearest POP, private backbone)              | Not always (may not enter Azure at nearest edge)              |
| Session Failover Speed           | Fast (in-path health checks, instant reroute)           | Slower (DNS TTL, cache delay)                                 |
| DNS Complexity                   | Single A record per Broker                              | ATM profile replaces A record, points to regional IPs         |
| Client Experience                | Consistent, seamless                                    | May experience more variable latency, slow failover           |
| Azure Dependency                 | Requires global public IP inventory                     | No Global IP required                                         |
| Scalability                      | Native Anycast, easy glob. expansion                    | Add regional IPs to ATM profile                               |
| Data Path                        | CR-LB is in the data path (L4 proxy)                    | ATM is NOT in data path (DNS only)                           |

---

## 7. Summary and Guidance

- **Traffic Manager** enables TCP (and any protocol) traffic routing by DNS, with health checks, weighted or performance routing, and is widely available.
- **ATM provides DNS resolution only**—it returns the best regional endpoint IP, then the client connects directly to that IP.
- **Clients (Controller/Target)** connect to the resolved regional LB endpoint returned by ATM, which may not always be the absolute lowest-latency Azure entry point, and traffic will not traverse the Azure backbone if entering a far region.
- **CR-LB remains preferred** for global single-IP, nearest POP entry and optimal backbone routing. But in scenarios where Global Standard Public IPs are not available (inventory exhaustion, region limitations), **Traffic Manager is a practical fallback**.
- If using ATM, always ensure your Standard LBs have healthy and routable public IPs, and monitor DNS TTL for faster failover/fallback scenarios.

---
