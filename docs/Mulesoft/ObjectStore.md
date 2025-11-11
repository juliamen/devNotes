# MuleSoft Object Store Guide

## 1️⃣ Overview
The **Object Store** in MuleSoft is a key–value storage mechanism used to persist data across Mule flows or application executions.  
It enables data storage between executions, Mule components, or different Mule apps depending on configuration: in-memory, persistent, or external.

---

## 2️⃣ What Is an Object Store?
- Stores objects (Strings, JSON, serialized Java objects) with a **unique key**.  
- Acts like a small database for temporary or persistent data.

**Common Uses:**
- Caching API responses
- Tracking batch job progress
- Managing scheduler locks
- Storing tokens or timestamps
- Deduplication or debounce logic

---

## 3️⃣ Why Use an Object Store?

| Use Case         | Description                                         |
|-----------------|-----------------------------------------------------|
| State Management | Track variables/timestamps across executions      |
| Deduplication   | Avoid processing duplicate messages                |
| Scheduler Control | Store last run time or lock flags                  |
| Caching         | Reduce latency by caching API/DB results           |
| Token Management | Store API tokens until expiration                 |
| Resiliency      | Persist data across app restarts                   |

---

## 4️⃣ Object Store Types

| Type                   | Description                     | Lifetime               | Example Use                       |
|------------------------|---------------------------------|----------------------|----------------------------------|
| In-Memory              | JVM memory                     | Lost after restart    | Scheduler flags, transient cache |
| Persistent             | Stored on disk                 | Survives restart      | Timestamps, tokens               |
| Cloud Object Store v2  | Managed by CloudHub            | Survives redeploy     | Production persistence           |
| Custom                 | External (Redis, DB, etc.)    | Depends on impl.      | Enterprise-scale caching         |

---

## 5️⃣ Object Store Connector Operations

### 1. Store
```xml
<os:store key="lastRunTimestamp" objectStore="SchedulerStore">
    <os:value><![CDATA[#[vars.currentTimestamp]]]></os:value>
</os:store>
````

* Save timestamps, tokens, or calculated values.

### 2. Retrieve

```xml
<os:retrieve key="lastRunTimestamp" objectStore="SchedulerStore">
    <os:default-value><![CDATA[#['1900-01-01 00:00:00.000']]]></os:default-value>
</os:retrieve>
```

* Fetch stored data; return default if key missing.

### 3. Contains

```xml
<os:contains key="token" objectStore="AuthStore" />
```

* Conditional logic:

```xml
<choice>
  <when expression="#[!payload]">
    <!-- Generate new token -->
  </when>
  <otherwise>
    <!-- Use cached token -->
  </otherwise>
</choice>
```

### 4. Remove

```xml
<os:remove key="token" objectStore="AuthStore" />
```

* Delete expired tokens or stale data.

### 5. Clear

```xml
<os:clear objectStore="SchedulerStore" />
```

* Reset store completely (use carefully in production).

---

## 6️⃣ Defining an Object Store

**In-Memory Example:**

```xml
<os:object-store name="SchedulerStore" persistent="false" />
```

**Persistent Example:**

```xml
<os:object-store name="SchedulerStore" persistent="true" entryTtl="0" />
```

**Attributes:**

| Attribute          | Description                             |
| ------------------ | --------------------------------------- |
| persistent         | true = stored on disk, survives restart |
| entryTtl           | Time-to-live (ms); 0 = never expire     |
| maxEntries         | Maximum number of entries               |
| expirationInterval | How often expired entries are cleaned   |

---

## 7️⃣ CloudHub Object Store v2

* Fully managed, persistent, per-app isolation
* Scalable for production workloads
* Accessible via Anypoint Runtime Manager UI/API

**Note:** Each worker has isolated store unless configured as shared.

---

## 8️⃣ Common Use Cases and Patterns

| Use Case                | Example                              |
| ----------------------- | ------------------------------------ |
| Scheduler Control       | Last execution timestamp             |
| Token Cache             | API access token until expiry        |
| Request Deduplication   | Store message IDs for 24h            |
| Batch Progress Tracking | Record offsets for partial restarts  |
| Temporary Caching       | DB query results                     |
| Locking Mechanism       | Key as lock indicator for scheduling |

---

## 9️⃣ Example: Scheduler with Persistent Timestamp

```xml
<flow name="DailyJobFlow">
    <scheduler frequency="1d" timeZone="Europe/London" />
    
    <os:retrieve key="lastRunTimestamp" objectStore="SchedulerStore">
        <os:default-value><![CDATA[#['1900-01-01 00:00:00.000']]]></os:default-value>
    </os:retrieve>

    <logger message="Last run: #[payload]" level="INFO" />

    <!-- Processing logic -->

    <set-variable variableName="currentTimestamp" value="#[now() as String {format: 'yyyy-MM-dd HH:mm:ss.SSS'}]" />
    <os:store key="lastRunTimestamp" objectStore="SchedulerStore">
        <os:value><![CDATA[#[vars.currentTimestamp]]]></os:value>
    </os:store>
</flow>
```

**Outcome:**

* Retrieves last run time
* Executes logic
* Updates timestamp
* Value persists after redeploy

---

## 10️⃣ Integration with Azure Pipeline

* Object Store config remains within the Mule app
* Store names, TTLs, keys can be externalized

```yaml
variables:
  - group: MuleSoft-Dev

steps:
  - task: Bash@3
    script: |
      mvn deploy \
        -Dmule.env=dev \
        -DobjectStoreName=$(objectStoreName)
```

Mule XML:

```xml
<os:object-store name="${objectStoreName}" persistent="true" />
```

---

## 11️⃣ Best Practices

| Category        | Recommendation                                                 |
| --------------- | -------------------------------------------------------------- |
| Persistence     | Use `persistent="true"` for critical data                      |
| Key Naming      | Meaningful, unique keys (`jobName:lastRun`, `token:${system}`) |
| Expiration      | Set `entryTtl` for transient caches                            |
| Size Management | Configure `maxEntries` to prevent unbounded growth             |
| Security        | Avoid storing sensitive data unless encrypted                  |
| Error Handling  | Use `<os:default-value>` when retrieving keys                  |
| Cleanup         | Remove/clear keys as part of maintenance/testing               |

---

## 12️⃣ Troubleshooting

| Issue                     | Likely Cause         | Fix                                  |
| ------------------------- | -------------------- | ------------------------------------ |
| Value lost after redeploy | Store not persistent | Set `persistent="true"`              |
| Key not found             | Key expired/removed  | Use default value on retrieve        |
| Object store full         | maxEntries exceeded  | Increase limit or clear old entries  |
| Unexpected sharing        | Same store name used | Use unique names per app/environment |

---

## 13️⃣ Summary Table

| Connector Operation | Purpose               | Common Use             |
| ------------------- | --------------------- | ---------------------- |
| `<os:store>`        | Save data under a key | Timestamps, tokens     |
| `<os:retrieve>`     | Fetch stored value    | Last run data          |
| `<os:contains>`     | Check key existence   | Token cache validation |
| `<os:remove>`       | Delete specific entry | Token invalidation     |
| `<os:clear>`        | Clear entire store    | Reset job or cache     |

---

## 14️⃣ Visual Summary

```
                ┌───────────────────────────┐
                │     Mule Object Store     │
                │ (in-memory / persistent)  │
                └───────────┬───────────────┘
                            │
      ┌─────────────────────┼─────────────────────┐
      ▼                     ▼                     ▼
 [os:store]           [os:retrieve]          [os:contains]
 Save key-value        Get by key             Check key exists
      │                     │                     │
      ▼                     ▼                     ▼
 [os:remove]           [os:clear]            [Scheduler Flow]
 Delete entry           Reset store           Use timestamp
```

**Key Takeaways:**

* Object Store = key-value persistence mechanism
* In-memory → temporary; persistent → survives restarts
* CloudHub OSv2 → persistent by default
* Use for tokens, timestamps, caching, deduplication, and state tracking
* Externalize store names/settings for CI/CD integration

