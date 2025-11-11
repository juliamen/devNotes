
# MuleSoft Environment Variables Guide

## 1️⃣ Overview

Environment Variables in MuleSoft are **externalized configuration parameters** that allow developers to write environment-agnostic Mule applications.

Instead of hardcoding URLs, credentials, or other environment-specific values, you define them as variables supplied at runtime depending on the deployment environment (Dev, Test, UAT, Prod).

**Benefits include:**

- Secure configuration management
- Seamless CI/CD pipeline automation
- Promoting the same Mule application artifact across environments

---

## 2️⃣ Why Use Environment Variables?

| Benefit               | Description                                                                 |
|-----------------------|-----------------------------------------------------------------------------|
| Environment Isolation  | Different environments can have different endpoints, credentials, or Object Store names without changing code |
| Security              | Sensitive values (passwords, tokens) aren’t stored in version control        |
| Portability           | Single deployable JAR can run in any environment                             |
| Automation            | Pipelines (Azure DevOps, Jenkins) can inject values dynamically             |
| Consistency           | Prevents configuration drift between environments                            |

---

## 3️⃣ Common Use Cases

| Example              | Variable                  | Example Value                        |
|---------------------|---------------------------|-------------------------------------|
| API endpoint         | api.host                  | https://api.dev.company.com          |
| Database connection  | db.user, db.password      | db_user, db_pass123!                 |
| Salesforce credentials | sf.username, sf.password, sf.token | Salesforce integration credentials |
| Azure Key Vault endpoint | azure.keyvault.url      | https://my-keyvault.vault.azure.net/|
| Logging level        | log.level                 | INFO / DEBUG                         |

---

## 4️⃣ How MuleSoft Reads Environment Variables

### Option A — Config.properties + Secure.properties
```properties
# src/main/resources/config.properties
api.host=${env.API_HOST}
api.key=${env.API_KEY}
db.user=${env.DB_USER}
````

`${env.API_HOST}` reads the environment variable `API_HOST`.

### Option B — Runtime Manager (CloudHub)

* Runtime Manager → Application → Settings → Properties tab

```text
API_HOST=https://api.dev.company.com
DB_USER=devuser
DB_PASSWORD=secret
```

* Mule resolves `${env.VARIABLE_NAME}` automatically.

### Option C — On-Prem Mule / Mule Wrapper

* Define in `wrapper.conf`:

```text
wrapper.java.additional.10=-DAPI_HOST=https://api.dev.company.com
wrapper.java.additional.11=-DDB_USER=devuser
```

* Use in Mule as `${API_HOST}`

---

## 5️⃣ Environment Variables vs Secure Properties

| Aspect    | Environment Variables                    | Secure Properties (File Encryption) |
| --------- | ---------------------------------------- | ----------------------------------- |
| Purpose   | Runtime configuration                    | Protect sensitive info at rest      |
| Storage   | System environment or deployment config  | Encrypted `.properties` files       |
| Best for  | CI/CD pipelines, environment differences | Secrets (passwords, tokens)         |
| Used with | `${env.VAR_NAME}`                        | `${secure::keyName}`                |

> **Tip:** Combine them — e.g., Azure pipeline sets decryption key as environment variable for Mule runtime.

---

## 6️⃣ Example: Usage in Mule Flow

```xml
<http:listener-config name="HTTP_Listener_config"
    host="${api.host}"
    port="${api.port}" />

<db:config name="DB_Config">
    <db:connection>
        <db:generic-connection
            url="${db.url}"
            user="${db.user}"
            password="${db.password}" />
    </db:connection>
</db:config>
```

* Placeholders automatically resolved at runtime.

---

## 7️⃣ How They Fit Into Azure Pipeline

### YAML Example

```yaml
trigger:
  branches:
    include:
      - main

variables:
  - group: MuleSoft-Dev   # Variable group with environment values

stages:
  - stage: DeployToDev
    displayName: "Deploy to Dev"
    jobs:
      - job: Deploy
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: Maven@3
            inputs:
              goals: 'clean package'
          - task: Bash@3
            displayName: "Deploy to CloudHub"
            inputs:
              targetType: inline
              script: |
                echo "Deploying Mule app..."
                mvn deploy -Dmule.env=dev \
                  -Danypoint.username=$(anypointUsername) \
                  -Danypoint.password=$(anypointPassword) \
                  -DAPI_HOST=$(apiHost) \
                  -DDB_USER=$(dbUser) \
                  -DDB_PASSWORD=$(dbPassword)
```

**Explanation:**

* Variable Group (`MuleSoft-Dev`) defines environment-specific values.
* Variables injected as system properties (`-DAPI_HOST`) → Mule resolves as `${API_HOST}`.

---

## 8️⃣ Azure Variable Groups Example

| Variable Group | Key        | Value                                                      | Secret |
| -------------- | ---------- | ---------------------------------------------------------- | ------ |
| MuleSoft-Dev   | apiHost    | [https://api.dev.company.com](https://api.dev.company.com) | ❌      |
| MuleSoft-Dev   | dbUser     | dev_db_user                                                | ❌      |
| MuleSoft-Dev   | dbPassword | ********                                                   | ✅      |

> Each environment (Dev, UAT, Prod) can have its own variable group.

---

## 9️⃣ Best Practices

✅ **DO**

* Use environment variables for all environment-specific config
* Keep secrets in Azure Key Vault or secure variables
* Name variables consistently (`API_HOST`, `DB_USER`, `DB_PASSWORD`)
* Reference variables in `config.properties` to avoid XML clutter
* Maintain a `.env.sample` for local dev reference

🚫 **DON’T**

* Hardcode values in Mule XML or properties files
* Commit secrets/tokens to version control
* Use separate code branches for environment-specific configs — rely on variables instead

---

## 10️⃣ Summary Diagram

```
                ┌────────────────────────┐
                │   Azure Pipeline YAML   │
                │------------------------│
                │ Variable Group: Dev     │
                │   API_HOST=...          │
                │   DB_USER=...           │
                └──────────┬─────────────┘
                           │
                           ▼
              ┌────────────────────────────┐
              │  Maven Build & Deploy Task │
              │  -DAPI_HOST=$(API_HOST)    │
              │  -DDB_USER=$(DB_USER)      │
              └──────────┬─────────────────┘
                           │
                           ▼
             ┌────────────────────────────┐
             │   Mule Runtime / CloudHub  │
             │  Uses ${API_HOST}, ${DB_USER} │
             └────────────────────────────┘
```

---

## ✅ Summary Table

| Concept              | Purpose                 | Where Defined           | Mule Usage          |
| -------------------- | ----------------------- | ----------------------- | ------------------- |
| Environment Variable | Externalized config     | OS / Pipeline / Runtime | `${env.VAR_NAME}`   |
| System Property      | Runtime argument        | `-DVAR=value`           | `${VAR}`            |
| Secure Property      | Encrypted config file   | `.properties` + key     | `${secure::key}`    |
| Azure Variable Group | Central pipeline config | Azure DevOps Library    | Injected into build |

