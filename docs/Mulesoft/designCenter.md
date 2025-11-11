
# MuleSoft Design Center: Reusable Data Library & API Spec Guide

## 1. MuleSoft Project Structure in Design Center

| Project Type       | Example Name          | Purpose                                     |
|-------------------|---------------------|---------------------------------------------|
| API Specification  | Customer API         | Defines endpoints/resources/methods        |
| Library (Fragment) | Common Data Types Library | Defines reusable types, examples, traits, resourceTypes, etc. |

---

## 2. Create a Data Type Library in Design Center

### Step 2.1 — Create the Library
- Design Center → Create → New Fragment → Library  
- Name: `common-data-library`  
- RAML 1.0 format

### Step 2.2 — Folder Structure

```

/types
CustomerType.raml
AddressType.raml
/examples
CustomerExample.json
AddressExample.json
library.raml

````

### Step 2.3 — Define Reusable Data Types

**CustomerType.raml**
```raml
#%RAML 1.0 DataType
type: object
properties:
  id: string
  name: string
  email: string
  address:
    type: Address
````

**AddressType.raml**

```raml
#%RAML 1.0 DataType
type: object
properties:
  street: string
  city: string
  postalCode: string
```

### Step 2.4 — Create Example Payloads

**CustomerExample.json**

```json
{
  "id": "12345",
  "name": "John Doe",
  "email": "john@example.com",
  "address": {
    "street": "123 High Street",
    "city": "London",
    "postalCode": "E1 6AN"
  }
}
```

### Step 2.5 — Link Types & Examples in Root Library

**library.raml**

```raml
#%RAML 1.0 Library
usage: Common reusable data types and examples

types:
  Address: !include types/AddressType.raml
  Customer: !include types/CustomerType.raml

examples:
  customerExample: !include examples/CustomerExample.json
```

### Step 2.6 — Publish the Library

* Click **Publish to Exchange** → give version (e.g., 1.0.0) → Publish
* Available for import into any API project

---

## 3. Create Your Main API Specification

### Step 3.1 — Create API Spec

* Design Center → Create → New API Specification → `Customer API` → RAML 1.0

### Step 3.2 — Folder Structure

```
/api
  customer.raml
/examples
  postCustomerExample.json
api.raml
```

### Step 3.3 — Import Data Library

**api.raml**

```raml
#%RAML 1.0
title: Customer API
version: v1
baseUri: https://api.example.com/{version}
mediaType: application/json

uses:
  common: exchange_modules/<org-id>/common-data-library/1.0.0/library.raml

/customer:
  get:
    description: Get customer details
    responses:
      200:
        body:
          application/json:
            type: common.Customer[]
            example: common.customerExample
  post:
    body:
      application/json:
        type: common.Customer
        example: !include examples/postCustomerExample.json
    responses:
      201:
        body:
          application/json:
            type: common.Customer
```

**Note:** `exchange_modules/<org-id>/common-data-library/1.0.0/library.raml` is auto-inserted when you **Add dependency → Library → Select from Exchange**.

---

## 4. Using Reusable Traits or Resource Types (Optional)

**traits/pagination.raml**

```raml
#%RAML 1.0 Trait
queryParameters:
  page:
    type: integer
    description: Page number
  pageSize:
    type: integer
    description: Number of results per page
```

**Usage in API**

```raml
uses:
  common: exchange_modules/<org-id>/common-data-library/1.0.0/library.raml

/customer:
  get:
    is: [ common.pagination ]
```

---

## 5. Folder Best Practices

| Folder           | Purpose                                 |
| ---------------- | --------------------------------------- |
| /types           | DataType definitions (for reuse)        |
| /examples        | Example JSON payloads                   |
| /traits          | Common reusable query/header patterns   |
| /resourceTypes   | Shared structures for similar resources |
| /schemas         | JSON Schema definitions                 |
| /securitySchemes | OAuth2, Client ID, Basic Auth           |

---

## 6. Publish to Exchange

* Click **Publish → Publish to Exchange**
* Provide version & visibility (private/public)
* API and library become discoverable

---

## 7. Manage the API in API Manager

### Step 7.1 — Create API Instance

* API Manager → Manage API → Create new API → Select from Exchange
* Choose API version, environment (Sandbox/Dev/Prod)
* Define **Autodiscovery ID** for Mule app

### Step 7.2 — Apply Policies

* Client ID Enforcement
* Rate Limiting / SLA Tiers
* CORS
* OAuth 2.0 Access Token Enforcement
* Logging

**Note:** Autodiscovery allows API Gateway to enforce policies without code changes

---

## 8. Implement the API in Anypoint Studio

1. Open Studio → File → New → Mule Project
2. Import from Exchange → select API
3. Scaffold flows from RAML → APIkit router generated

**Example Generated Flow**

```xml
<flow name="get:/customer:api-config">
  <logger message="Fetching customers..." level="INFO" />
  <set-payload value='#[{ id: "1", name: "John Doe" }]' />
</flow>
```

**Add Autodiscovery in `mule-artifact.json`**

```json
{
  "apiId": "1234567",
  "groupId": "org-id",
  "artifactId": "customer-api",
  "version": "1.0.0"
}
```

---

## 9. Governance & Versioning Tips

| Practice                                      | Why                                       |
| --------------------------------------------- | ----------------------------------------- |
| Use central library for common types & traits | Keeps specs consistent & maintainable     |
| Version library & API specs semantically      | Safe evolution (1.0.0 → 1.1.0 → 2.0.0)    |
| Don’t copy/paste examples                     | Reduce duplication, import from library   |
| Publish first → link via `uses:`              | Manages dependencies                      |
| Use API Manager policies                      | Separation of concerns, avoid custom code |

---

## 10. Visual Reuse Flow

```
+--------------------------+
| Common Data Library      |
|  - Types (Customer)      |
|  - Examples (JSON)       |
|  - Traits (pagination)   |
+-----------+--------------+
            |
            | uses:
            v
+--------------------------+
| Customer API             |
|  - api.raml              |
|  - resources/            |
|  - examples/             |
+-----------+--------------+
            |
            | Published to Exchange
            v
+--------------------------+
| API Manager / Gateway    |
|  - Apply Policies        |
|  - Analytics & SLA       |
+--------------------------+
```

---

## ✅ Summary Checklist

| Step | Action                                 | Done |
| ---- | -------------------------------------- | ---- |
| 1    | Create common data library (fragment)  | ☐    |
| 2    | Add types, examples, traits            | ☐    |
| 3    | Publish library to Exchange            | ☐    |
| 4    | Create main API spec                   | ☐    |
| 5    | Import library (`uses:`)               | ☐    |
| 6    | Define resources and link examples     | ☐    |
| 7    | Publish API spec to Exchange           | ☐    |
| 8    | Manage in API Manager & apply policies | ☐    |
| 9    | Import API into Studio & implement     | ☐    |
| 10   | Add autodiscovery and deploy           | ☐    |


