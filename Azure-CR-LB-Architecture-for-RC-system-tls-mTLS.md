# TLS & mTLS Architecture FAQ  
### For RC System Using Azure CR‑LB + Standard LBs  


This FAQ explains how TLS and mTLS are implemented in the RC system architecture using Azure Cross-Region Load Balancer (CR‑LB) and Standard LBs. It covers certificate placement, sample commands, renewal flow, and how the FQDN maps to the certificates.

---

#  1. Do we bind SSL/TLS certificates on CR‑LB or Standard LB?

**No.**  
Azure Cross‑Region Load Balancer (CR‑LB) and Standard Load Balancer operate strictly at **Layer 4 (TCP)**. They do **not** support:

- SSL termination  
- Certificate binding  
- SNI routing  
- TLS offloading  
- HTTPS inspection  

**Encrypted traffic simply passes through to the Broker unchanged.**

TLS/mTLS terminates **ONLY on the Broker instance**.

---

#  2. Where are certificates installed?

###  Certificates are installed *on the Broker instances* 

Each Broker has its own unique server certificate matching its FQDN:

- broker-us-1.company.com  
- broker-us-2.company.com  
- broker-in-3.company.com  
- broker-in-4.company.com  

Types of certificates needed:

| Certificate | Used By | Purpose |
|-------------|---------|---------|
| **Server Certificate** | Broker | Authenticates Broker to Controller/Target |
| **Client Certificate** | Controller & Target | Authenticates clients to Broker (mTLS) |
| **CA Certificate / Intermediate CA** | All | Trust validation for mTLS |

**CN/SAN must match the Broker hostname.**

---

#  3. Where should certificates be stored?

###  Use **Azure Key Vault (per region)**

Each Broker VM/Pod retrieves from the local-region Key Vault:

- Server Certificate (PFX)  
- CA Chain  
- Private key  
- Renewal policy  

### Access method:
- **Managed Identity** (no secrets, no passwords)

---

#  4. How does mTLS work in this architecture?

### Step 1 — Client initiates TLS  
Controller/Target opens outbound TCP 443 → CR‑LB → Standard LB → Broker.

### Step 2 — Broker presents its server certificate  
Client validates:
- CN/SAN matches Broker FQDN  
- Certificate is trusted  
- Not expired  

### Step 3 — Client presents certificate  
Broker validates:
- Client certificate signed by your CA  
- Certificate not expired/blocked  

### Step 4 — Secure mTLS session established  
LBs do not inspect the packets.

---

#  5. Does the LB need the Broker certificate?

**No.**  
LBs are transparent TCP routers.

---

#  6. Certificate File Formats

- Broker server certificates → **PFX** (certificate + private key)
- Client certificates → **PFX**
- CA bundle → **CRT** or **PEM**

---

#  7. Sample: Import Broker Cert to Key Vault

```bash
az keyvault certificate import   --vault-name kv-us   --name broker-us-1-cert   --file broker-us-1.pfx   --password "<pfx-password>"
```

---

#  8. Sample: VMSS or any other way of Broker Access to Cert

### Assign Managed Identity

```bash
az vmss identity assign   --resource-group rg-us   --name vmss-b1-us
```

### Grant Key Vault permissions

```bash
az keyvault set-policy   --name kv-us   --object-id <MSI_OBJECT_ID>   --certificate-permissions get list
```

---

#  9. Certificate Renewal / Rotation Process

### Step 1 — Upload new cert to Key Vault
```bash
az keyvault certificate import   --vault-name kv-us   --name broker-us-1-cert   --file new-cert.pfx
```

### Step 2 — Update VMSS extension or AKS Secret Store  
Broker loads new cert on restart or via signal.

### Step 3 — Drain and roll instances
```bash
az vmss update-instances   --instance-ids <ids>   --resource-group rg-us   --name vmss-b1-us
```

---

#  10. Do we need SAN entries for CR‑LB IPs?

**No.**

TLS validation uses the hostname used by the client.
CR‑LB IP is internal routing only.

Example:

```
Client connects to: broker-us-1.company.com
LB forwards traffic → Broker B1
Broker presents cert for broker-us-1.company.com
```

No IP-based SAN needed.

---

#  11. Do we need separate certificates per Broker?

**Yes.**  
Each Broker exposes its own hostname → each requires a certificate:

- broker-us-1.company.com → cert1  
- broker-us-2.company.com → cert2  
- broker-in-3.company.com → cert3  
- broker-in-4.company.com → cert4  

---

#  12. Do we configure SSL on CR-LB or Standard LB?

**No. 100% unnecessary.**

Both are TCP pass‑through devices.

Only the **Broker** handles TLS.

---

#  13. Example Broker Certificate Configuration

For `broker-in-3.company.com`:

```
Subject CN = broker-in-3.company.com
SAN = DNS:broker-in-3.company.com
Issuer = Your Private CA
Extended Key Usage = Server Authentication
Key Size = 2048 or 4096
```

---

#  14. TLS Flow Diagram

<img width="749" height="118" alt="image" src="https://github.com/user-attachments/assets/ba154130-bc22-432f-9495-205999e22a67" />


LBs simply route TCP packets.

---

#  15. Summary 

- CR‑LB and Standard LBs **do not** terminate SSL → correct by design  
- TLS/mTLS terminates on the **Broker** only  
- Broker certificates stored in **Key Vault**  
- Broker presents cert matching its specific FQDN  
- Controller/Target present client cert for mTLS  
- LBs forward encrypted traffic without inspecting it  
- No cert binding needed on any LB  

---


