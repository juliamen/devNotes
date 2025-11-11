
# MuleSoft API Gateway & Autodiscovery Guide

## 1️⃣ Overview

In MuleSoft, the **API Gateway** and **Autodiscovery** enable centralized API management — applying policies, security, analytics, and monitoring without changing application logic.

When you deploy an API-enabled Mule application, **Autodiscovery** links the runtime (CloudHub or on-prem) to the API instance in **Anypoint API Manager**.

**Benefits:**

- Centralized policy enforcement (OAuth, Rate Limiting, IP Allowlist)
- API analytics and usage monitoring
- Security and governance consistency
- Simplified lifecycle management (Design → Implement → Manage → Monitor)

---

## 2️⃣ What is the MuleSoft API Gateway?

### Definition
- Embedded runtime component in Mule 4+
- Intercepts all API traffic and applies governance/security policies configured in API Manager

### Key Responsibilities

| Function             | Description                                           |
|---------------------|-------------------------------------------------------|
| Policy Enforcement   | Access control, rate limiting, client ID enforcement |
| Analytics & Monitoring | Captures metrics, usage data, latency for dashboards |
| Request Routing      | Directs API requests to backend flows/services       |
| Security Layer       | Auth & authorization before API logic executes       |
| Lifecycle Management | Versioning, status tracking, deprecation handling   |

### Architecture Overview

```

┌────────────────────────────────────────────┐
│             Anypoint API Manager           │
│   (Policies, Contracts, Analytics, Version) │
└───────────────┬────────────────────────────┘
│ Autodiscovery ID
▼
┌──────────────────────────────┐
│        Mule API Gateway       │
│  (Runtime inside Mule App)    │
│ Applies policies             │
│ Enforces security            │
│ Sends analytics to API Manager │
└──────────────────────────────┘

````

---

## 3️⃣ What is API Autodiscovery?

**Definition:** Mechanism linking a Mule API implementation (Mule app) to its API instance in API Manager.

- Ensures the runtime enforces all policies and sends analytics to API Manager.
- Uses a unique **Autodiscovery ID** per API instance.

---

## 4️⃣ How Autodiscovery Works

- Mule reads Autodiscovery configuration from XML or property
- Connects to Anypoint Platform using credentials
- Registers runtime as active API instance
- API Gateway begins applying policies and analytics automatically

---

## 5️⃣ Autodiscovery Configuration in Mule

### Step 1 — Get the Autodiscovery ID
- API Manager → Manage API → Settings → **API Autodiscovery ID**

### Step 2 — Configure in Mule App

**Inline in XML:**
```xml
<api-gateway:autodiscovery
    api-id="12345678"
    flow-ref="api-main"
    http-listener-config="HTTP_Listener_config"/>
````

**Using Property Variable:**

```properties
# config.properties
api.autodiscovery.id=12345678
```

```xml
<api-gateway:autodiscovery
    api-id="${api.autodiscovery.id}"
    flow-ref="api-main"
    http-listener-config="HTTP_Listener_config"/>
```

---

## 6️⃣ Required Prerequisites

| Requirement                 | Description                                          |
| --------------------------- | ---------------------------------------------------- |
| API Registered in Exchange  | RAML/OAS published to Exchange                       |
| API Instance in API Manager | Created in API Manager with Autodiscovery ID         |
| Connected Mule Runtime      | CloudHub or on-prem linked to Anypoint control plane |
| Proper Credentials          | Client ID/Secret configured for platform access      |

---

## 7️⃣ Configuring Autodiscovery Across Environments

| Environment     | How It’s Configured                               | Notes                                         |
| --------------- | ------------------------------------------------- | --------------------------------------------- |
| CloudHub        | Runtime Manager properties or CI/CD pipeline      | -Danypoint.platform.client_id / client_secret |
| On-Prem Runtime | wrapper.conf or deployment script                 | Connects via Anypoint control plane           |
| Azure DevOps    | Inject Autodiscovery ID and credentials at deploy | Enables automatic registration during CI/CD   |

---

## 8️⃣ Example: Azure Pipeline Integration

```yaml
variables:
  - group: MuleSoft-UAT

steps:
  - task: Bash@3
    displayName: "Deploy API with Autodiscovery"
    inputs:
      targetType: inline
      script: |
        mvn clean deploy \
          -Danypoint.username=$(anypointUsername) \
          -Danypoint.password=$(anypointPassword) \
          -Danypoint.platform.client_id=$(anypointClientId) \
          -Danypoint.platform.client_secret=$(anypointClientSecret) \
          -Dapi.autodiscovery.id=$(autodiscoveryId) \
          -Dmule.env=uat
```

* Autodiscovery ID and credentials are passed as variables
* Runtime registers with API Manager → policies automatically applied

---

## 9️⃣ Policies and Governance

| Policy                | Purpose                                         |
| --------------------- | ----------------------------------------------- |
| Client ID Enforcement | Ensures requests provide valid client ID/secret |
| OAuth 2.0             | Auth via external IdP                           |
| Rate Limiting         | Controls requests per time unit                 |
| IP Allowlist/Denylist | Restricts certain IP ranges                     |
| Header Injection      | Adds tracking/security headers automatically    |

**Note:** Policies are managed centrally in API Manager — no code changes needed in Mule flows.

---

## 🔒 Example End-to-End Flow

1. API published to Exchange
2. API instance created in API Manager → Autodiscovery ID generated
3. Mule app configured with `${api.autodiscovery.id}`
4. Deployed via CI/CD → runtime registers with API Manager
5. Client calls API → Gateway enforces security → backend flow executes
6. Analytics/metrics synced to API Manager dashboard

---

## 10️⃣ Verifying Autodiscovery

✅ Runtime Manager Logs

* Look for: `API Gateway: Successfully registered API with ID 12345678`

✅ API Manager

* Status: Active
* Connected Environment matches deployment
* Policies applied
* Analytics visible

---

## 11️⃣ Best Practices

| Category      | Best Practice                                                             |
| ------------- | ------------------------------------------------------------------------- |
| Configuration | Externalize Autodiscovery ID per environment                              |
| Security      | Never hardcode credentials; use pipeline vars or secure properties        |
| Automation    | Integrate Autodiscovery and credentials into CI/CD                        |
| Governance    | Apply policies centrally in API Manager                                   |
| Monitoring    | Enable API Analytics for all production APIs                              |
| Versioning    | Version APIs in Exchange & API Manager; avoid reusing IDs across versions |

---

## 12️⃣ Troubleshooting Tips

| Issue                | Likely Cause                     | Fix                                      |
| -------------------- | -------------------------------- | ---------------------------------------- |
| API not “Active”     | Wrong Autodiscovery ID           | Check API Manager ID and property config |
| Policies not applied | Missing credentials / wrong env  | Verify Anypoint client ID/secret         |
| Analytics missing    | API Manager not connected        | Ensure connectivity and correct org/env  |
| Unauthorized request | Policy enforcement not satisfied | Provide valid credentials                |

---

## 13️⃣ Summary Table

| Concept              | Description                                | Defined Where            | Purpose                             |
| -------------------- | ------------------------------------------ | ------------------------ | ----------------------------------- |
| API Gateway          | Embedded Mule component enforcing policies | Mule Runtime             | Security, monitoring, routing       |
| Autodiscovery        | Links Mule app to API Manager              | Mule XML + API Manager   | Policy management, analytics        |
| Autodiscovery ID     | Unique ID of API instance                  | API Manager              | Connects runtime to API definition  |
| Platform Credentials | Client ID/Secret for registration          | Azure or Runtime Manager | Authenticate runtime to API Manager |
| Policies             | Rules applied by API Gateway               | API Manager              | Enforce security and governance     |

---

## 14️⃣ Visual Summary Diagram

```
                   ┌───────────────────────────────────────────────┐
                   │           Anypoint API Manager                │
                   │ Policies | Analytics | Versions | Contracts   │
                   └──────────────────────┬────────────────────────┘
                                          │ Autodiscovery ID
                                          ▼
                             ┌─────────────────────────────┐
                             │     Mule Runtime (Gateway)  │
                             │ Enforces policies           │
                             │ Registers API instance      │
                             └──────────┬──────────────────┘
                                        │
                                        ▼
                          ┌───────────────────────────────┐
                          │     Mule Application Flows    │
                          │   (Business Logic / APIs)     │
                          └───────────────────────────────┘
```

