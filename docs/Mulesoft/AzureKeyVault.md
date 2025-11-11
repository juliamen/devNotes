# Azure Key Vault Properties Provider – MuleSoft Configuration Guide

## 1. Overview

This guide explains how to configure **MuleSoft’s Azure Key Vault Properties Provider** so that your Mule application retrieves secrets directly from **Azure Key Vault at runtime — before the application starts.**

This allows secure storage of sensitive values (usernames, passwords, API keys) without hardcoding them into Mule configuration files.

### ✅ Key Benefits

- Secrets are **never stored in plain text** in Mule config.
- Mule retrieves secrets from Azure Key Vault automatically at startup.
- Works seamlessly with Mule placeholder syntax `${propertyName}`.
- Supported across all connectors (Salesforce, DB, HTTP, etc.).

---

## 2. Prerequisites

| Requirement | Description |
|-------------|-------------|
| Anypoint Runtime Version | Mule 4.4+ or Mule 4.5+ |
| Connector Version | Azure Key Vault Properties Provider v2.1+ |
| Azure Access | Existing Azure Key Vault |
| Credentials | Service Principal (Client ID + Secret) with access to Key Vault |
| MuleSoft Access | Ability to add dependencies (Studio or Maven) |

---

## 3. High-Level Architecture

```

Mule App → Azure Key Vault Properties Provider → Azure Key Vault
(resolves ${...} placeholders)          (stores secrets)

````

**Runtime behavior**

1. Mule loads config files (config.yaml, global.xml)
2. When it sees a placeholder (e.g., `${sf.username}`), it calls Azure Key Vault
3. The resolved secret is injected before the app starts

---

## 4. Step 1: Add the Azure Key Vault Properties Provider

### ✅ Option A – Anypoint Studio

1. Open your Mule project  
2. Go to **Global Elements → Create → Search: _Azure Key Vault Properties Provider_**
3. Select **Azure Key Vault Properties Provider (Properties Placeholder)**
4. Click OK and configure fields (see Step 2)

### ✅ Option B – Maven Dependency

Add to `pom.xml`:

```xml
<dependency>
    <groupId>com.mulesoft.connectors</groupId>
    <artifactId>mule-azure-key-vault-properties-provider</artifactId>
    <version>2.1.0</version>
    <classifier>mule-plugin</classifier>
</dependency>
````

---

## 5. Step 2: Configure the Provider

**Global config example:**

```xml
<azure-keyvault-properties-provider:config name="Azure_KeyVault_Config"
    doc:name="Azure Key Vault Properties Provider"
    client-id="${azure.client.id}"
    client-secret="${azure.client.secret}"
    tenant-id="${azure.tenant.id}"
    vault-name="${azure.vault.name}"
    key-vault-url="https://${azure.vault.name}.vault.azure.net/"
    connection-timeout="30000"
    read-timeout="30000" />
```

### Parameter Description

| Parameter          | Description                                          |
| ------------------ | ---------------------------------------------------- |
| client-id          | Azure App Registration (Service Principal) client ID |
| client-secret      | Service principal client secret                      |
| tenant-id          | Azure AD tenant ID                                   |
| vault-name         | Name of the Azure Key Vault                          |
| key-vault-url      | Vault URL (e.g., `https://myvault.vault.azure.net/`) |
| connection-timeout | (Optional) Connection timeout (ms)                   |
| read-timeout       | (Optional) Read timeout (ms)                         |

These can be placed in `mule-artifact.properties` or secure external config.

---

## 6. Step 3: Store Secrets in Azure Key Vault

1. Go to **Azure Portal**
2. Navigate to:
   **Key Vaults → [Your Vault] → Secrets → Generate/Import**
3. Create secrets for Mule

Example:

| Azure Secret Name | Value                                                           |
| ----------------- | --------------------------------------------------------------- |
| `sf.username`     | [salesforce_user@domain.com](mailto:salesforce_user@domain.com) |
| `sf.password`     | P@ssw0rd123                                                     |
| `sf.token`        | a1b2c3d4e5f6                                                    |

The secret names must **match the Mule placeholder keys exactly**.

---

## 7. Step 4: Reference Secrets in Mule

You can use them like normal `${...}` placeholders.

### ✅ Salesforce Config Example

```xml
<sfdc:config name="Salesforce_Config"
             doc:name="Salesforce Configuration"
             username="${sf.username}"
             password="${sf.password}"
             securityToken="${sf.token}"
             authentication="BASIC">
    <sfdc:connection host="login.salesforce.com"/>
</sfdc:config>
```

Mule will:

1. Read `${sf.username}`, `${sf.password}`, `${sf.token}`
2. Query Azure Key Vault
3. Inject values before startup

---

## 8. Step 5: Verify the Integration

Deploy to **CloudHub**, **RTF**, or **local runtime**.

Check logs for secret retrieval:

```
INFO  [Azure Key Vault Properties Provider] Successfully retrieved secret 'sf.username' from vault 'mykeyvault'
```

Then confirm the connector initializes successfully.

---

## 9. Optional: Multiple Environment Support

Use separate vaults or name patterns.

### Example vaults

* `mykeyvault-dev`
* `mykeyvault-prod`

### Example secret naming

* `sf.username.dev`
* `sf.username.prod`

Reference dynamically:

```xml
username="${sf.username.${mule.env}}"
```

In `mule-artifact.properties`:

```
mule.env=dev
```

---

## 10. Troubleshooting

| Issue              | Cause                  | Resolution                                        |
| ------------------ | ---------------------- | ------------------------------------------------- |
| Secret not found   | Wrong secret name      | Ensure secret name matches Mule placeholder       |
| App fails to start | Invalid credentials    | Check service principal and tenant ID             |
| 403 Forbidden      | Missing access policy  | Add Key Vault access policy with Get permissions  |
| Timeout            | Network/firewall issue | Ensure Mule runtime can reach `*.vault.azure.net` |

---

## 11. Security Best Practices

✅ Use **Managed Identities** instead of client secrets when possible
✅ Grant **least privilege** access in Key Vault
✅ Rotate secrets regularly
✅ Do **not** commit credentials to source control
✅ Enable **Azure Monitor** for Key Vault access logging

---

## 12. Reference Links

* MuleSoft Azure Key Vault Properties Provider v2.1 Docs
* Azure Key Vault Documentation
* MuleSoft Secure Property Placeholder Best Practices

```

