
# Salesforce ↔ MuleSoft Bidirectional Integration Guide

## 0) Quick Prerequisites

- Anypoint Platform account with access to:
  - Design Center
  - Exchange
  - API Manager
  - Target Environment
- Anypoint Studio (matching Mule runtime, e.g., Mule 4.x)
- Salesforce org (dev/sandbox) for Connected App
- Credentials/keys for external REST endpoint (and any required authentication)

---

## 1) Design the API (RAML) in Design Center

1. **Create API Spec:**  
   - Log in → Design Center → Create → New API Specification  
   - Choose **RAML 1.0**, give name/version, create project  

2. **Define API:**  
   - Add Data Types, Resources, Methods, query/path params, examples, responses  
   - Add `securitySchemes` for OAuth2, client-credentials, JWT if needed  
   - Use visual or RAML editor; create reusable fragments (types, traits)  

3. **Example Responses:**  
   - Add at least one example response per method for mocking & scaffolding  

4. **Enable Mocking Service (optional):**  
   - Public or private mock endpoints for early consumer testing  

**Minimal RAML Example:**

```raml
#%RAML 1.0
title: Integration API
version: v1
baseUri: https://api.example.com/{version}
mediaType: application/json

securitySchemes:
  oauth_2_0:
    type: OAuth 2.0
    describedBy:
      headers:
        Authorization:
          description: "Bearer access token"
          type: string
      responses:
        401:
          description: Unauthorized
    settings:
      accessTokenUri: https://auth.example.com/oauth/token
      authorizationGrants: [ client_credentials ]

types:
  Case:
    type: object
    properties:
      Id: string
      Subject: string
      Status: string

/cases:
  get:
    securedBy: [ oauth_2_0 ]
    description: Retrieve cases
    responses:
      200:
        body:
          application/json:
            type: Case[]
  post:
    securedBy: [ oauth_2_0 ]
    body:
      application/json:
        type: Case
    responses:
      201:
        body:
          application/json:
            type: Case
````

---

## 2) Publish RAML to Anypoint Exchange

* Open API Designer → **Publish → Publish to Exchange**
* Choose semantic version, description, visibility
* Enables discoverability and use in Studio / API Manager
* Test the API spec & mock service in Exchange

---

## 3) Import RAML into Anypoint Studio and Scaffold Flows

**Options:**

### A — Import from Exchange (recommended)

1. File → New → Mule Project → Import a published API → From Exchange
2. Select API/version → finish
3. Studio imports RAML into `src/main/resources/api` and generates scaffold

### B — Import Local RAML

1. Save RAML locally (`src/main/resources/api/api.raml`)
2. File → New → Mule Project → right-click RAML → **Generate Flows from Local REST API** (APIkit)

**Notes about scaffolding:**

* APIkit generates one flow per resource/method + API Router
* Use generated flows as skeleton: add HTTP listener, connectors, transformations
* Re-generate when spec changes (Studio prompts)

---

## 4) Implement Bidirectional Integration (High-Level)

### A. Salesforce → Mule → External REST

1. **Event trigger:**

   * Platform Events (recommended), Change Data Capture, Outbound Messages, or scheduled jobs
   * Choose based on volume & consistency

2. **Mule Flow:**

   ```
   HTTP Listener / Salesforce Connector inbound → Transform Message (DataWeave) → HTTP Request / REST Connector
   ```

3. **Retries & Idempotency:**

   * Use `until-successful` / queue (Anypoint MQ / persistent VM)
   * Avoid duplicate processing with idempotency key
   * Connector reconnection: apply strategy where supported (HTTP Request requires explicit retries)

### B. External REST → Mule → Salesforce

* Expose endpoints via RAML / API → secure with API Manager policies
* Flow: receive request → validate & transform → Salesforce Connector to create/update or publish Platform Events
* Protect/throttle with Client ID, rate limiting, JWT, etc.

---

## 5) Configure Salesforce Auth & Connector

* Create **Connected App** in Salesforce

  * OAuth 2.0 JWT Bearer recommended for server-to-server
  * Upload public cert, configure scopes & pre-approval

* Anypoint Studio:

  * Add Salesforce Connector → configure global connector (JWT or other OAuth/credentials)
  * Store consumer key / keystore / alias in **Secure Configuration Properties**

**Important:** never hardcode credentials

---

## 6) Data Mapping Example (DataWeave)

```dw
%dw 2.0
output application/json
---
{
  caseId: payload.Id,
  subject: payload.Subject,
  status: payload.Status,
  submittedAt: payload.CreatedDate as String
}
```

---

## 7) Error Handling, Retries, Testing

* **Error Handling:** use `error-handler`, `on-error-continue`, `on-error-propagate`
* **Retries:** until-successful or queue-driven for idempotent ops

  * Connector reconnection strategies for DB/JMS; HTTP Request needs explicit retries
* **Automated Tests:** MUnit for flows; MUnit Test Recorder for complex flows

---

## 8) Governance, Security & Policies

* Publish API → create API instance in **API Manager** → attach runtime (Autodiscovery)
* Apply policies: Client ID enforcement, OAuth token, rate limiting, JWT validation
* Policies are runtime-enforced, no code change

---

## 9) Deployment & CI/CD / Observability

* Deploy to CloudHub, Runtime Fabric, or on-prem Mule runtime
* Use Maven / Anypoint CLI / Jenkins / GitHub Actions for CI/CD
* Enable Anypoint Monitoring & alerts; integrate logs with aggregation tools
* API versioning: semantic versioning, Exchange versions for consumers

---

## 10) Quick Checklists

* [ ] RAML defines types, examples, security schemes
* [ ] Publish spec to Exchange; enable mock service
* [ ] Import API to Studio / generate flows via APIkit
* [ ] Configure Salesforce Connector (JWT for server-to-server); secure secrets
* [ ] Apply API Manager policies; enable Autodiscovery
* [ ] Add MUnit tests & mocks; create CI/CD pipeline

---

## Useful References

* [Publish API Spec from Design Center → Exchange](https://docs.mulesoft.com)
* [Import / Generate flows from RAML in Anypoint Studio (APIkit)](https://docs.mulesoft.com)
* [Mocking Service (Design Center)](https://docs.mulesoft.com)
* [Secure Configuration Properties](https://docs.mulesoft.com)
* [Salesforce Connector & JWT Flow](https://docs.mulesoft.com)
* [API Manager & Policies](https://docs.mulesoft.com)


