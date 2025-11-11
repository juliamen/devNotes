
# Salesforce & On-Premise Integration using MuleSoft VPS & VPN

## 1. Overview

This document outlines the high-level design for integrating **Salesforce** with **on-premise systems** using MuleSoft’s **Virtual Private Space (VPS)** and **VPN**.

- **VPN** provides a secure, encrypted tunnel between MuleSoft and the on-premise network  
- **VPS** provides an isolated, secure deployment environment  
- APIs inside VPS communicate with Salesforce and on-prem systems such as:
  - **OpenHousing**
  - **IntelliPrint**

---

## 2. Architecture Overview

| Component | Description |
|-----------|-------------|
| **MuleSoft Virtual Private Space (VPS)** | Dedicated, isolated network to deploy secure Mule APIs |
| **VPN Connection** | Secure tunnel between VPS and on-prem systems |
| **Salesforce Integration** | Mule APIs interact with Salesforce using connectors |
| **On-Premise Systems** | Example systems: OpenHousing, IntelliPrint |

---

## 3. Setting Up the Private Space

### ✅ Create Private Space

1. Navigate to **Anypoint Platform → Runtime Manager → Private Spaces**
2. Click **Create Private Space**

### ✅ Configure Networking

In the **Firewall Rules** tab:

- **Ingress Rules** – allow inbound traffic from trusted networks
- **Egress Rules** – allow outbound traffic only to required systems

### ✅ Create Private Network & Configure CIDR Block

- **Region:** Select deployment region  
  *(Note: Some accounts enforce a default region that cannot be changed)*

- **CIDR Block:** Defines the IP range of the Private Space

**CIDR guidance:**
- MuleSoft recommends **/22** → 1,024 IPs
- Allowed range: **/24 (256 IPs)** → **/16 (65,536 IPs)**
- Example: `192.168.0.0/16`

Choosing the correct CIDR prevents IP conflicts with the customer’s on-prem networks.

---

## 4. Establish Secure VPN Connectivity

### ✅ Create VPN Connection

1. In the Private Space, go to **Network → Create Connection**
2. Select **VPN**
3. Enter a **Connection Name**
4. Enter the **Remote IP Address**
   - This is the customer’s public IP of their VPN gateway
   - Could be a firewall, router, or VPN appliance

### ✅ Choose Routing Type

#### **A) Dynamic (BGP)**
- Requires ASN configuration
- MuleSoft automatically assigns **169.254.x.x** link-local addresses for BGP sessions

#### **B) Static Routing**
If using static routing, define:

1. **On-Premise Network CIDR**
   - e.g. `172.16.0.0/16`

2. **MuleSoft VPC CIDR**
   - e.g. `10.1.0.0/22`

**Example Static Route:**

| Destination | Required By |
|-------------|-------------|
| `172.16.0.0/16` | Configure in MuleSoft VPN |
| `10.1.0.0/22` | Configure on on-prem VPN side |

### ✅ Local Point-to-Point IP

| Routing Type | Is 169.254.x.x needed? | Why |
|--------------|-----------------------|-----|
| **Static** | ❌ Not required | Routes defined manually |
| **BGP** | ✅ Required | Used for dynamic route advertisements |

If static routing is used but 169.254.x.x appears, it’s a MuleSoft default — it will not affect routing.

### ✅ Configure On-Prem VPN Gateway

- Download the **VPN configuration guide** from Anypoint Platform
- Apply the settings to the on-prem device
- Ensure correct encryption, hashing, tunnel lifetimes, and routing

### ✅ Configure Firewall Rules

- Allow traffic between MuleSoft VPS and on-prem network
- Make sure necessary ports/protocols (IPSec) are permitted

---

## 5. Deploying APIs in the Private Space

### ✅ Deployment Steps

1. In **Runtime Manager**, deploy your Mule application
2. Select the Private Space as the **deployment target**
3. Configure runtime, workers, and environment variables

### ✅ Internal Communication

- APIs inside VPS can communicate using **internal DNS**
- Traffic stays entirely within the secure environment

### ✅ Load Balancing & Scaling

- Enable **load balancing** to distribute traffic
- Enable **auto-scaling** to adjust instances based on demand

---

## 6. Security & Access Controls

| Control Layer | Purpose |
|----------------|---------|
| **Firewall Rules** | Restrict inbound access to trusted IPs |
| **Egress Controls** | Limit outbound traffic to approved services |
| **Anypoint Security Policies** | Apply policies such as: <br> - Client ID Enforcement <br> - IP Whitelisting <br> - Rate Limiting <br> - TLS/HTTPS enforcement |

---

## 7. Summary

Using VPS + VPN provides:

✅ Isolated secure API hosting  
✅ Encrypted connection to on-prem systems  
✅ No public exposure of APIs  
✅ Internal DNS + private networking  
✅ Scalable and enterprise-grade architecture  

---

## 8. Additional Reference

For more information about creating and configuring Virtual Private Spaces:  
*MuleSoft Anypoint Platform → Private Spaces Documentation*


