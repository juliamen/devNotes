
# SQL Pagination in MuleSoft Using Database Connector & DataWeave

## 🎯 Objective

When querying large datasets from a SQL database, pagination prevents:

- High memory usage
- Timeouts or oversized payloads
- Network bottlenecks

This guide explains how to design a MuleSoft flow that retrieves records in pages (batches) using **LIMIT / OFFSET** with the Database Connector and DataWeave.

---

## ⚙️ Prerequisites

- Anypoint Studio **7.12+**
- Mule Runtime **4.x**
- Database Connector configured
- A database table with many records (e.g., `Customer`, `Account`)

---

## 🧩 Pagination Strategy Overview

SQL pagination pattern:

```sql
SELECT * FROM customer
ORDER BY id
LIMIT #[vars.pageSize] OFFSET #[vars.offset];
````

| Variable   | Meaning                                                      |
| ---------- | ------------------------------------------------------------ |
| `pageSize` | Number of records per page                                   |
| `offset`   | Number of rows to skip (e.g., `(pageNumber - 1) * pageSize`) |

You keep looping until fewer than `pageSize` records are returned.

---

## 🏗️ Example Flow Design

**Flow Name:** `get-customers-paginated`
**Purpose:** Retrieve all customers from DB in chunks (100 per call) and aggregate into a single result list.

---

### 🔹 Step 1: Initialize Variables

```xml
<flow name="get-customers-paginated">
    <set-variable variableName="pageSize" value="100" doc:name="Set Page Size"/>
    <set-variable variableName="offset" value="0" doc:name="Set Offset"/>
    <set-variable variableName="hasMore" value="true" doc:name="Has More Flag"/>
    <set-variable variableName="allResults" value="#[]" doc:name="Init Result Array"/>
```

---

### 🔹 Step 2: Loop Until No More Records

```xml
<while expression="#[vars.hasMore]" doc:name="While - Paginate">
```

#### ✅ Database Select (paginated query)

```xml
<db:select doc:name="Select Paged Customers" config-ref="DB_Config">
    <db:sql>
        <![CDATA[
        SELECT * FROM customer
        ORDER BY id
        LIMIT :pageSize OFFSET :offset
        ]]>
    </db:sql>
    <db:input-parameters>
        <db:input-parameter key="pageSize" value="#[vars.pageSize]"/>
        <db:input-parameter key="offset" value="#[vars.offset]"/>
    </db:input-parameters>
</db:select>
```

#### ✅ Check if more records exist

```xml
<choice doc:name="Check Result Size">
    <when expression="#[sizeOf(payload) < vars.pageSize]">
        <set-variable variableName="hasMore" value="false" doc:name="Set hasMore False"/>
    </when>
    <otherwise>
        <set-variable variableName="offset" value="#[vars.offset + vars.pageSize]" doc:name="Increase Offset"/>
    </otherwise>
</choice>
```

#### ✅ Accumulate results

```xml
<set-variable variableName="allResults" value="#[vars.allResults ++ payload]" doc:name="Append to Results"/>
</while>
```

---

### 🔹 Step 3: Transform Final Output (Optional)

```xml
<ee:transform doc:name="Transform Final Output">
    <ee:message>
        <ee:set-payload><![CDATA[
            %dw 2.0
            output application/json
            ---
            {
                totalRecords: sizeOf(vars.allResults),
                data: vars.allResults
            }
        ]]></ee:set-payload>
    </ee:message>
</ee:transform>
</flow>
```

---

## 🧠 How It Works

* Variables control pagination
* While loop repeats until the last page
* Each iteration:
  ✅ Queries database with LIMIT / OFFSET
  ✅ Checks if results are smaller than `pageSize`
  ✅ Appends results to array
  ✅ Increases offset for next loop
* Final output returns combined dataset

---

## 🧪 Example Output

```json
{
  "totalRecords": 245,
  "data": [
    { "id": 1, "name": "Alice" },
    { "id": 2, "name": "Bob" }
  ]
}
```

---

## 🛡️ Best Practices

| Area           | Recommendation                                                     |
| -------------- | ------------------------------------------------------------------ |
| Performance    | Use `ORDER BY` on indexed columns                                  |
| Memory         | For huge datasets, process records per page instead of storing all |
| Error Handling | Wrap DB call in a Try scope                                        |
| Config         | Externalize `pageSize` in `config.properties`                      |
| Large scale    | Use Batch Jobs for very large datasets                             |

---

## 🧰 Optional: Using Batch Job (Alternative Approach)

If you intend to process the records (e.g., write to Salesforce), use Batch:

```xml
<batch:job name="customer-batch-job">
  <batch:input>
    <db:select config-ref="DB_Config" fetchSize="500">
      <db:sql>SELECT * FROM customer</db:sql>
    </db:select>
  </batch:input>
  ...
</batch:job>
```

Batch automatically handles pagination via `fetchSize`.

---

