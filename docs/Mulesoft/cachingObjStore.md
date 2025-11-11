
# MuleSoft Storage Options Guide: Object Store vs Cache vs Database

## 1️⃣ Overview
MuleSoft provides several ways to store or cache data.  
Choosing the right option depends on **data lifetime, volume, persistence, sharing, and performance**.

The main options:

- **Object Store** – key-value persistence for small/medium state data
- **Caching Module** – temporary in-memory cache for request/response payloads
- **Database Storage** – long-term, transactional storage

---

## 2️⃣ Comparison Table

| Capability           | Object Store                 | Caching Module           | Database Storage           |
|---------------------|-----------------------------|-------------------------|---------------------------|
| Purpose             | Key-value persistence        | Temporary cache         | Long-term transactional   |
| Data Model          | Key-Value (simple map)      | Key-Value (in-memory)   | Relational / NoSQL        |
| Persistence          | Optional (persistent=true / OSv2) | No (in-memory)     | Yes (disk-based / external) |
| Scope               | App-specific (can be shared)| Within one app/worker   | Enterprise-wide           |
| Performance         | Very fast                   | Fast (memory)           | Slower (DB/network)       |
| Scalability         | Good per Mule app           | Limited (memory bound)  | Excellent (DB scaling)    |
| Survives Restart    | ✅ If persistent            | ❌ Lost on restart      | ✅ Always                 |
| CloudHub Use        | Cloud Object Store v2       | Works per worker        | External DB required      |
| Data Size Limit     | Small/Medium (MBs)          | Small                   | Large (GBs/TBs)           |
| Access              | Mule connectors (os:)       | Caching scope (cache:)  | Mule DB Connector (db:)   |
| Concurrency Control | Basic (no transactions)     | None                    | Full ACID support         |
| Backup & Recovery   | Basic / Cloud-managed       | None                    | Provided by DB engine     |
| Security            | Local / platform-managed    | Memory only             | DB-level control          |
| Typical TTL         | Configurable (entryTtl)     | Short (cache)           | Permanent or DB-managed   |
| Cost / Overhead     | Lightweight                 | Very lightweight        | Heavier (external dep.)   |

---

## 3️⃣ When to Use Each

| Scenario                       | Recommended Option                  | Reason                                      |
|--------------------------------|------------------------------------|--------------------------------------------|
| Track last run timestamp       | 🟩 Object Store (Persistent)       | Lightweight, survives redeploy             |
| Cache API responses             | 🟨 Caching Module                  | High-speed memory cache, short TTL         |
| Store API tokens/session data   | 🟩 Object Store (Persistent/OSv2) | Fast, survives restarts                     |
| Deduplicate incoming events     | 🟩 Object Store                     | Store message IDs for a retention period   |
| Long-term storage / reporting   | 🟥 Database Storage                 | Handles large volumes, permanent           |
| Transactional updates / audit   | 🟥 Database Storage                 | Data integrity                              |
| Temporary enrichment data       | 🟨 Caching Module                  | Ultra-fast, non-persistent                 |
| Cross-application coordination  | 🟩 Cloud Object Store (OSv2 Shared)| Accessible by multiple Mule apps           |

---

## 4️⃣ Example Usage by Layer (API-Led Architecture)

| API Layer      | Common Use                         | Recommended Storage                 |
|----------------|------------------------------------|------------------------------------|
| System API     | Persist token for external system  | Object Store (persistent)           |
| Process API    | Deduplication / state checkpoint  | Object Store (persistent)           |
| Experience API | Cache UI data / Salesforce lookup | Caching Module (in-memory)         |
| Data Layer/ETL | Store transformed data, logs       | Database (SQL / NoSQL)             |

---

## 5️⃣ Quick Decision Matrix

| Question                                           | If Yes → Use                  |
|---------------------------------------------------|-------------------------------|
| Do you need data to survive redeployments?       | 🟩 Object Store (persistent)  |
| Need ultra-fast, temporary storage?              | 🟨 Caching Module             |
| Need relational queries or reports?              | 🟥 Database                   |
| Share state across multiple Mule apps?           | 🟩 Cloud Object Store (OSv2)  |
| Need transaction rollback or joins?              | 🟥 Database                   |
| Only local caching for a flow / connector call?  | 🟨 Caching Module             |

---

## 6️⃣ Example Patterns

### 🟩 Object Store – Token Cache
```xml
<flow name="GetAuthTokenFlow">
    <os:retrieve key="apiToken" objectStore="AuthStore">
        <os:default-value><![CDATA[#['']]]></os:default-value>
    </os:retrieve>

    <choice>
        <when expression="#[isEmpty(payload)]">
            <logger message="Token not found — generating new one" />
            <set-variable variableName="newToken" value="#[dw::HTTP.request(...).body.token]" />
            <os:store key="apiToken" objectStore="AuthStore">
                <os:value><![CDATA[#[vars.newToken]]]></os:value>
            </os:store>
        </when>
        <otherwise>
            <logger message="Token found in Object Store" />
        </otherwise>
    </choice>
</flow>
````

### 🟨 Caching Module – Repeated API Call

```xml
<flow name="GetCustomerDataFlow">
    <cache:scope>
        <http:request method="GET" url="https://api.example.com/customers" />
    </cache:scope>
</flow>
```

* Result: Cached API response reused within TTL window.

### 🟥 Database – Persistent Storage

```xml
<flow name="LogRequestFlow">
    <db:insert>
        <db:sql><![CDATA[
            INSERT INTO request_logs (id, timestamp, status)
            VALUES (:id, :timestamp, :status)
        ]]></db:sql>
    </db:insert>
</flow>
```

* Result: Data stored permanently for audits or analytics.

---

## 7️⃣ Summary: Choosing the Right Tool

| Need                                | Use This                  | Why                           |
| ----------------------------------- | ------------------------- | ----------------------------- |
| Persist lightweight key-value pairs | Object Store              | Simple, built-in, reliable    |
| Cache results for quick reuse       | Caching Module            | High-speed, transient         |
| Store & query structured data       | Database                  | Full persistence and querying |
| Maintain scheduler states / tokens  | Object Store (Persistent) | Retains across restarts       |
| Share cache/state across apps       | Cloud Object Store v2     | Managed and scalable          |

---

## ✅ Key Takeaways

* **🟩 Object Store** → Best for small, persistent, key-value state or metadata.
* **🟨 Caching Module** → Best for temporary, fast, memory-level caching.
* **🟥 Database** → Best for structured, durable, transactional data.
* **CloudHub Object Store v2** → Persistent by default; survives redeploys without configuration.
* Always design around **data volume, lifetime, and access pattern** when choosing storage.

