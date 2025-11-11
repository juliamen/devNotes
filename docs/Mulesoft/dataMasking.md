
# Data Masking & Conditional Logging in MuleSoft (DataWeave 2.0)

## 📘 Overview

This document explains how to implement **data masking** in MuleSoft using **DataWeave 2.0**, and how to control logging using a global property (`enableLogging`).  
Sensitive fields are automatically masked with `*****` before being logged or sent to external systems.

---

## 🧩 1. Prerequisites

- Mule Runtime **4.x**
- **DataWeave 2.0**
- Global configuration property defined in `config.properties`

Example:

```properties
# config.properties
enableLogging=true
````

---

## 🧠 2. DataWeave Script Example

```dw
%dw 2.0
output application/json
import * from dw::util::Values

var enableLogging = p('enableLogging') default false

fun maskData(data) =
    data
        mask "Asset_Type_VC" with "*****"
        mask "Other_Asset_Reference_VC" with "*****"
        mask "Number_of_Bedrooms_SI" with "*****"

---
if (enableLogging) 
    maskData(payload)
else 
    "Logging disabled: Sensitive data not shown"
```

---

## 🧱 3. Explanation

| Component            | Description                                           |
| -------------------- | ----------------------------------------------------- |
| `mask`               | DataWeave operator used to mask fields in the payload |
| `"*****"`            | Replacement masking value                             |
| `enableLogging`      | Boolean global property used to turn logging on/off   |
| `p('enableLogging')` | Reads property from Mule config/environment           |
| `maskData(payload)`  | Function that masks defined sensitive fields          |

---

## ⚙️ 4. Global Property Configuration

You can define `enableLogging` from:

* `config.properties`
* Environment variable
* Runtime Manager → Application properties

Example (Runtime Manager override):

```bash
-DenableLogging=false
```

---

## 🧾 5. Example Input

```json
{
  "Asset_Type_VC": "Flat",
  "Other_Asset_Reference_VC": "A12345",
  "Number_of_Bedrooms_SI": "2",
  "Postcode": "SW1A 1AA"
}
```

---

## ✅ 6. Example Output (Logging Enabled)

```json
{
  "Asset_Type_VC": "*****",
  "Other_Asset_Reference_VC": "*****",
  "Number_of_Bedrooms_SI": "*****",
  "Postcode": "SW1A 1AA"
}
```

---

## 🚫 7. Example Output (Logging Disabled)

```json
"Logging disabled: Sensitive data not shown"
```

---

## 🧩 8. Integration with Logger Component

Add the DataWeave transformation before the logger to ensure masking takes place:

```xml
<set-payload value="#[dw('classpath:dw/maskData.dwl')]" doc:name="Mask Sensitive Fields"/>
<logger level="INFO" message="#[payload]" doc:name="Log Masked Payload"/>
```

---

## 🔍 9. Best Practices

✅ Always mask PII and sensitive data before logging
✅ Use `enableLogging=false` in Production
✅ Use `enableLogging=true` only in Development/Testing
✅ Store masking logic in a reusable DWL file (e.g., `/src/main/resources/dw/maskData.dwl`)

---

