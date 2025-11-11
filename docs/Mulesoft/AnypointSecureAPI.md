
# Anypoint API Manager – Secure API Management Guide

## 1. Overview

This guide describes how to:

- Create and manage **API instances** in Anypoint API Manager  
- Apply policies (Client ID Enforcement, Rate Limiting, etc.)
- Configure **Auto-Discovery** in Mule applications for runtime policy enforcement
- Register **Connected Apps** for secure authentication
- Manage **Contracts** for consumer access

These steps ensure that MuleSoft APIs are **secure**, **discoverable**, and only accessible to **authorized clients**.

---

## 2. Prerequisites

Before starting, ensure:

- ✅ You have Anypoint Platform access with API Manager + Exchange permissions  
- ✅ The Mule app is deployed to CloudHub or Runtime Fabric  
- ✅ A Connected App is created in Access Management  
- ✅ You have an API Auto-Discovery ID from API Manager

---

## 3. Step-by-Step Implementation

### ✅ Step 1: Publish API Specification to Exchange

1. Design API specification in **Design Center** (RAML or OAS 3.0)
2. Validate using the **API Console**
3. Publish to Exchange:
   - Click **Publish → Publish to Exchange**
   - Select a business group and set version details

Once published, the API becomes available to create managed instances in API Manager.

---

### ✅ Step 2: Create an API Instance in API Manager

1. Go to **Anypoint Platform → API Manager**
2. Click **Manage API → Manage API from Exchange**
3. Select your published API specification
4. Configure:
   - **Deployment Type:** CloudHub, RTF, or Hybrid
   - **Environment:** Sandbox, Production, etc.
   - **Implementation URI:** e.g. `https://myapi.cloudhub.io/api`
5. Click **Save & Manage API**

Your API instance is now created and ready for policies and contracts.

---

### ✅ Step 3: Apply Security Policies

Policies enforce authentication, throttling, logging, etc.

#### Example: Apply Client ID Enforcement Policy

1. In API Manager, open your API instance
2. Go to **Policies → Apply New Policy**
3. Choose **Client ID Enforcement**
4. Configure:
   - Client ID / Secret via **HTTP Headers** or **Query Parameters**
5. Save the policy

This ensures only registered clients can call the API.

---

### ✅ Step 4: Enable Auto-Discovery in Mule Application

Auto-Discovery allows the deployed Mule app to register itself with API Manager.

#### a. Retrieve Auto-Discovery ID

- In API Manager, open the API instance  
- Go to **Settings → Autodiscovery**
- Copy the API Instance ID

#### b. Add to Mule Configuration

Example (XML):

```xml
<api-platform-gw:api id="api-auto-discovery" apiId="12345678" flowRef="api-main-flow" />
````

Or configure via **Global Elements** (Mule 4):

* Add **API Autodiscovery**
* Paste API ID from API Manager

#### c. Deploy the Application

After deployment, the app automatically registers with API Manager and policies are enforced at runtime.

---

### ✅ Step 5: Create a Connected App for Secure Access

Connected Apps represent API consumers.

1. Go to **Access Management → Connected Apps**
2. Click **Create App**
3. Choose **Client Credentials**
4. Configure:

   * Name: `MyIntegrationApp`
   * Grant Type: `Client Credentials`
   * Scopes: assign access as needed
5. After creation, copy:

   * ✅ Client ID
   * ✅ Client Secret

These credentials will authenticate API calls.

---

### ✅ Step 6: Manage API Contracts

Contracts determine which apps may consume the API.

1. In **Exchange**, open the API asset
2. Click **Request Access**
3. Select:

   * API Instance
   * Application (Connected App)
4. After approval, a contract is established

The consumer now receives the Client ID + Secret needed to call the API.

---

### ✅ Step 7: Test the Secured API

Example using curl:

```bash
curl -X GET https://myapi.cloudhub.io/api/resource \
  -H "client_id: <your_client_id>" \
  -H "client_secret: <your_client_secret>"
```

* ✅ Successful request → valid response
* ❌ Missing/invalid values → `401` or `403`

---

## 8. Additional Security Enhancements

* Rate Limiting / Spike Control
* OAuth 2.0 policy for token-based access
* IP Whitelisting
* CORS policy
* TLS/HTTPS enforcement

---

## 9. Monitoring & Analytics

Once Auto-Discovery is enabled, API Manager provides:

* Request analytics (volumes, latency, failures)
* Policy violations
* Alerts & notifications via Anypoint Monitoring

---

## 10. Best Practices

✅ Always use Auto-Discovery for centralized governance
✅ Apply Client ID Enforcement at minimum
✅ Use Connected Apps + Contracts for consumer access control
✅ Version APIs and separate Sandbox vs Production
✅ Rotate client credentials regularly

