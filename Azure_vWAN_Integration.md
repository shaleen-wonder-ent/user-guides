# Context: Azure Virtual WAN Integration

## Question 1: Steps to connect with customer on-prem with already available Express Route if customer provides activation code

**Answer:**

To connect your customer's on-premises network to Azure Virtual WAN using an existing ExpressRoute circuit when they provide an activation code:

1. **Receive and Validate the Service Key**
   - The customer will provide you with an ExpressRoute circuit service key (authorization key) for an existing ExpressRoute circuit.

2. **Create ExpressRoute Gateway in Virtual WAN Hub**
   - Navigate to your Virtual WAN in the Azure Portal.
   - Select the virtual hub where you want to enable ExpressRoute connectivity.
   - Create an ExpressRoute gateway (if not already present).

3. **Add ExpressRoute Connection**
   - In the Virtual WAN hub, go to "ExpressRoute" connections.
   - Click "+ Add connection", enter the authorization key (service key) provided by the customer, configure routing settings.

4. **Validate Connection**
   - Check that the circuit status shows "Enabled" and "Provisioned".
   - Validate BGP peering is established.

**Points to note:**
- Ensure no overlapping address spaces between on-premises and Azure VNets.
- ExpressRoute circuits must be in the same geopolitical region (for standard SKU) or can be global (for premium SKU).
- Virtual WAN handles routing automatically between connected networks.

---

## Question 2: Connect this with our current hub – to start, we can connect using VPN

**Answer:**

To connect your Azure Virtual WAN hub to an existing traditional hub-and-spoke topology using VPN, there are 2 options:

**ER Style Diagram**

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/59a0b35a-ae6f-4af5-8a36-c105bb47774c" />

**Option 1 (Direct VNet Connection)**
- Azure Virtual WAN connects directly to VNet (no gateway allowed in VNet).
- ExpressRoute links on-prem to VWAN.
- ASN 65515 reserved for VWAN routing.

**Option 2 (Existing VPN Gateway)**
- Azure Virtual WAN connects via VPN to the existing VPN/ER Gateway in the hub VNet.
- On-prem uses ASN (custom, avoid 65515).
- This option supports gradual migration without downtime.

---

### **Option 1: Connect Existing VNet (without gateway) to Virtual WAN Hub**

**Prerequisites:**
- The existing hub VNet must **not** have a VPN Gateway or ExpressRoute Gateway.
- If a gateway exists, you'll need to remove it first (this will break existing connections temporarily).

**Steps:**
1. In Azure Portal, navigate to your Virtual WAN.
2. Select "Virtual network connections".
3. Click "+ Add connection".
4. Select your hub VNet.
5. Configure routing and propagation settings.
6. Click "Create".

---

### **Option 2: Connect Existing VPN Gateway to Virtual WAN (Recommended for Transition)**

If you have an existing VPN Gateway in your hub VNet and want to maintain connectivity during migration:

1. **Set Up VPN Gateway in Active-Active Mode**
   - Ensure your existing VPN Gateway is configured in active-active mode.
   - Note both public IP addresses of the gateway.

2. **Create VPN Sites in Virtual WAN**
   - In Virtual WAN, go to "VPN sites".
   - Create a new VPN site for each public IP of your gateway.
   - Configure the site with:
     - Device vendor information
     - Public IP addresses
     - Private address space (your hub VNet CIDR)
     - BGP settings (avoid using ASN 65515 – reserved for Virtual WAN)

3. **Create Site-to-Site VPN Gateway in Virtual WAN Hub**
   - Navigate to your Virtual WAN hub.
   - Create a VPN Gateway (Site-to-Site) if not already present.
   - Specify scale units based on throughput requirements.

4. **Connect VPN Sites to Hub**
   - Select "VPN (Site to site)" in your Virtual WAN hub.
   - Click "Connect VPN sites".
   - Select the VPN sites you created.
   - Configure connection settings:
     - Pre-shared key (PSK)
     - BGP settings
     - IPsec/IKE policy if needed

5. **Download VPN Configuration**
   - Download the VPN device configuration from Virtual WAN.
   - Apply this configuration to your existing VPN Gateway.

6. **Verify Connection**
   - Check connection status in Virtual WAN portal.
   - Validate routing propagation.
   - Test connectivity between resources.

**Points to note:**
- **Address Space:** Ensure no overlapping IP ranges between Virtual WAN hub, connected VNets, and on-premises networks.
- **Routing:** Virtual WAN provides automatic route propagation between all connected resources.
- **Transition Strategy:** Using VPN allows you to gradually migrate from traditional hub-and-spoke to Virtual WAN without downtime.
- **Cost:** Consider the cost implications of running both VPN Gateway and Virtual WAN during transition.

---

## References

- [Connect a VNet to a Virtual WAN hub – portal – Azure Virtual WAN (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/virtual-wan/howto-connect-vnet-hub)
- [Tutorial: Create site-to-site connections using Virtual WAN – Azure Virtual WAN (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-site-to-site-portal)
- [Connect a virtual network gateway to an Azure Virtual WAN (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/virtual-wan/connect-virtual-network-gateway-vwan)
- [Tutorial: Create an ExpressRoute association to Azure Virtual WAN (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-expressroute-portal)
- [About ExpressRoute Connections in Azure Virtual WAN (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-expressroute-about)
- [ExpressRoute documentation (Azure Docs)](https://docs.azure.cn/en-us/expressroute/)
