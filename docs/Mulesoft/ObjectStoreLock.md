
# MuleSoft Scheduler + CDC Pattern with Object Store Lock

## 📘 Purpose

This document explains how to:

- Prevent parallel executions of a scheduled MuleSoft job when a previous run hasn’t completed  
- Implement incremental data extraction (CDC) from SQL Server using a **timestamp watermark** pattern  

This ensures **data consistency**, **idempotent execution**, and no overlap between runs.

---

## 🧩 1. Overview of Components

| Component | Purpose |
|-----------|---------|
| Scheduler | Triggers Mule flow periodically (e.g., every 2 minutes) |
| Object Store (`SchedulerLockStore`) | Maintains lock and state (lock flag, last run timestamp) |
| Azure Key Vault | Stores DB credentials and API secrets securely |
| Database Connector | Queries SQL Server CDC view for incremental changes |
| HTTP Request | Pushes changed records to Salesforce Process API |

---

## 🛑 2. Preventing Parallel Execution

### 🎯 Objective

Ensure that only **one instance** of the scheduler runs at a time.  
If a job takes longer than its scheduled interval, the next run waits or skips.

### ⚙️ Mechanism: Object Store-Based Lock

- A flag (`isRunning`) is stored in an **Object Store** at the start of each run  
- If another instance finds the key already present, it skips execution  
- The flag is removed when the flow completes (success or error)

### 🔹 Implementation Steps

#### Step 1: Scheduler

```xml
<scheduler>
  <scheduling-strategy>
    <fixed-frequency frequency="5" timeUnit="MINUTES"/>
  </scheduling-strategy>
</scheduler>
````

#### Step 2: Acquire Lock

```xml
<os:store key="isRunning" objectStore="SchedulerLockStore" failIfPresent="true">
  <os:value>true</os:value>
</os:store>
```

> `failIfPresent="true"` ensures atomic lock creation. If the key exists, Mule throws `OS:KEY_ALREADY_EXISTS`.

#### Step 3: Skip if Locked

```xml
<on-error-continue type="OS:KEY_ALREADY_EXISTS">
  <logger level="INFO" message="Scheduler is still running, new run was skipped"/>
</on-error-continue>
```

#### Step 4: Release Lock

At the end of execution (success or failure):

```xml
<os:remove key="isRunning" objectStore="SchedulerLockStore"/>
```

#### 🔹 Error Handling Pattern

Wrap the flow in a try scope:

```xml
<on-error-propagate>
  <os:remove key="isRunning" objectStore="SchedulerLockStore"/>
  <logger level="ERROR" message="Flow failed. Lock released. Error: #[error.description]"/>
</on-error-propagate>
```

**Result:** Only one scheduler run executes at a time, preventing overlapping runs and DB contention.

---

## 🔄 3. Change Data Capture (CDC) Implementation

### 🎯 Objective

Extract **only new or changed records** since the last successful run.

### ⚙️ Mechanism: Timestamp Watermark Pattern

#### Step 1: Retrieve Last Run Timestamp

```xml
<os:retrieve key="lastAssetRunTimestamp" objectStore="SchedulerLockStore">
  <os:default-value>#['1900-01-01 00:00:00.000']</os:default-value>
</os:retrieve>
```

* Defaults to baseline date for first run
* Stored in Object Store for subsequent runs

#### Step 2: Build Incremental SQL Query

```sql
SELECT DISTINCT Asset_ID, ChangeTimestamp, ...
FROM [dbo].[Usr_SW_Salesforce_Assets_Changes_V]
WHERE ChangeTimestamp > '2025-10-08 00:00:00.000'
ORDER BY ChangeTimestamp ASC
OFFSET 0 ROWS FETCH NEXT 500 ROWS ONLY;
```

* `ChangeTimestamp` identifies new/updated records
* `OFFSET/FETCH` enables pagination

#### Step 3: Process and Push Records

* Mask sensitive fields (optional)
* Transform to JSON
* POST to Salesforce PAPI endpoint

#### Step 4: Capture Latest Change Timestamp

```xml
<ee:set-variable variableName="maxChangeTimestamp">
<![CDATA[%dw 2.0
output application/java
---
if (isEmpty(payload)) null
else (payload.ChangeTimestamp default [])
  reduce ((item, acc=max(item, acc)) -> 
      if (acc == null) item else if (item > acc) item else acc)
]]>
</ee:set-variable>
```

#### Step 5: Persist New Watermark

```xml
<os:store key="lastAssetRunTimestamp" objectStore="SchedulerLockStore">
  <os:value><![CDATA[#[[vars.maxChangeTimestamp as String {format: "yyyy-MM-dd HH:mm:ss.SSS"}]]]]></os:value>
</os:store>
```

> Ensures continuity for the next CDC run.

---

## 🧠 4. Control Flow Summary

| Step | Component          | Purpose                           |
| ---- | ------------------ | --------------------------------- |
| 1    | Scheduler          | Triggers periodic run             |
| 2    | Object Store Lock  | Prevents overlapping runs         |
| 3    | Retrieve Secrets   | Secure DB & API credentials       |
| 4    | CDC Query          | Get new/changed records           |
| 5    | Post to Salesforce | Send incremental data             |
| 6    | Update Timestamp   | Record latest processed timestamp |
| 7    | Remove Lock        | Release for next execution        |

---

## 🧱 5. Key Design Considerations

| Area             | Best Practice                                                             |
| ---------------- | ------------------------------------------------------------------------- |
| Object Store TTL | Set slightly longer than runtime to prevent orphan locks                  |
| Atomic Locking   | `failIfPresent=true` ensures exclusivity                                  |
| Fault Tolerance  | Wrap DB and HTTP calls in `until-successful`                              |
| Idempotency      | Downstream systems should tolerate duplicates if CDC replays              |
| Scalability      | Paginate using offset/limit for large datasets                            |
| Auditability     | Store run metadata (start/end times, record count) if traceability needed |

---

## 🧩 6. Example Runtime Behavior

| Time  | Action                | Result                                    |
| ----- | --------------------- | ----------------------------------------- |
| 00:00 | Scheduler triggers    | Lock acquired → starts                    |
| 00:01 | Second scheduler tick | Lock exists → skipped                     |
| 00:02 | Processing completed  | Lock removed                              |
| 00:03 | Next run starts       | Queries only records > last run timestamp |

---

## 📦 7. Benefits of This Pattern

* ✅ No duplicate processing — only new/changed records processed
* ✅ No overlapping executions — lock ensures single instance
* ✅ Recoverable — lock auto-expires; watermark persists
* ✅ Scalable and maintainable — suitable for nightly/hourly or near real-time syncs

---

## 🧾 8. Summary

| Aspect              | Implementation                                          |
| ------------------- | ------------------------------------------------------- |
| Locking Mechanism   | Object Store with `failIfPresent`                       |
| CDC Source          | SQL Server CDC / ChangeTimestamp view                   |
| State Management    | Object Store key for timestamp watermark                |
| Security            | Azure Key Vault for DB & API secrets                    |
| Error Handling      | Custom on-error scopes for lock cleanup & retries       |
| Integration Pattern | Scheduler → CDC Query → Transformation → Salesforce API |

---

