
# 🧭 MuleSoft Routing and Orchestration Patterns

## 1️⃣ Overview

MuleSoft provides a set of **routing and orchestration components** that control how messages move through your integration flows. These are essential for:

* ✅ Conditional processing
* ✅ Parallel processing
* ✅ Load balancing
* ✅ Aggregating responses

### Core Routing Components

| Router / Pattern                              | Description                                                         |
| --------------------------------------------- | ------------------------------------------------------------------- |
| **Choice Router**                             | Routes messages conditionally, like `if-else`.                      |
| **Scatter-Gather**                            | Executes multiple routes in parallel and aggregates results.        |
| **Round-Robin Router**                        | Distributes messages evenly across multiple routes.                 |
| **First Successful / Until Successful / All** | Implements failover, retries, or sequential multi-route processing. |

---

## 2️⃣ Choice Router

### 🧩 Definition

The **Choice Router** conditionally routes messages based on a **DataWeave expression** — similar to an `if-else-if` construct.

It:

* Evaluates conditions sequentially.
* Routes the message to the **first matching route**.
* Supports a final `otherwise` path as a default.

### 🔧 Syntax

```xml
<choice doc:name="Choice Router">
    <when expression="#[payload.type == 'A']">
        <logger message="Processing Type A" level="INFO"/>
    </when>
    <when expression="#[payload.type == 'B']">
        <logger message="Processing Type B" level="INFO"/>
    </when>
    <otherwise>
        <logger message="Processing default type" level="INFO"/>
    </otherwise>
</choice>
```

### 💡 Use Cases

* Routing based on payload values or headers
* Conditional execution of subflows
* Dynamic API processing (e.g., query parameter-based branching)

---

## 3️⃣ Scatter-Gather Router

### 🧩 Definition

**Scatter-Gather** allows parallel execution of multiple routes and then aggregates all responses into a single payload.

* Executes all routes **concurrently**
* Returns a **list of responses**

### 🔧 Syntax

```xml
<scatter-gather doc:name="Scatter-Gather">
    <flow-ref name="FlowA"/>
    <flow-ref name="FlowB"/>
</scatter-gather>
```

### 🧾 Example Output

```json
[
  { "FlowA": { ... } },
  { "FlowB": { ... } }
]
```

### 💡 Use Cases

* Parallel API calls
* Multi-system data enrichment
* Aggregating responses from multiple backends
* Reducing latency through concurrent processing

---

## 4️⃣ Round-Robin Router

### 🧩 Definition

**Round-Robin Router** distributes messages evenly across configured routes — useful for simple **load balancing** within a Mule application.

### 🔧 Syntax

```xml
<round-robin doc:name="Round Robin Router">
    <flow-ref name="Flow1"/>
    <flow-ref name="Flow2"/>
</round-robin>
```

### 💡 Use Cases

* Even task distribution across flows
* Load balancing between endpoints
* Ensuring balanced parallel processing

---

## 5️⃣ Other Routing Patterns

| Router / Component   | Description                                             | Use Case                         |
| -------------------- | ------------------------------------------------------- | -------------------------------- |
| **First Successful** | Sends message to multiple routes; returns first success | Failover handling                |
| **Until Successful** | Retries route until success or max attempts             | Retry transient failures         |
| **All**              | Executes all routes sequentially                        | Sequential multi-step operations |
| **Flow-Ref**         | Invokes another flow                                    | Modular subflow execution        |
| **Message Filter**   | Filters messages by condition                           | Pre-routing validation           |

---

## 6️⃣ Example – Combining Choice + Scatter-Gather

```xml
<flow name="MainFlow">
    <http:listener config-ref="HTTP_Listener_Config" path="/process"/>

    <choice doc:name="Route Based on Type">
        <when expression="#[payload.type == 'ASYNC']">
            <scatter-gather doc:name="Parallel Processing">
                <flow-ref name="AsyncFlowA"/>
                <flow-ref name="AsyncFlowB"/>
            </scatter-gather>
        </when>
        <otherwise>
            <logger message="Synchronous processing" level="INFO"/>
        </otherwise>
    </choice>
</flow>
```

### ✅ Explanation

* Routes messages of type `ASYNC` to multiple flows **in parallel**
* Other messages are handled **synchronously**

---

## 7️⃣ Combining Routers

| Combination                           | Purpose                          |
| ------------------------------------- | -------------------------------- |
| **Choice + Round-Robin**              | Conditional load balancing       |
| **Scatter-Gather + Until Successful** | Parallel processing with retries |
| **Choice + Flow-Ref**                 | Modular conditional routing      |

These combinations allow complex orchestration and fault-tolerant processing.

---

## 8️⃣ Best Practices

| Practice                                     | Reason                                      |
| -------------------------------------------- | ------------------------------------------- |
| Always include `<otherwise>` in Choice       | Prevents unhandled messages                 |
| Limit routes in Scatter-Gather               | Avoid CPU/memory overload                   |
| Use Round-Robin within the same runtime only | For cross-app, use VM or JMS                |
| Use Flow-Refs                                | Keep logic modular and maintainable         |
| Monitor Scatter-Gather payload size          | Large aggregations can impact memory        |
| Use appropriate error-handling               | Combine routers with `error-handler` scopes |

---

## 9️⃣ Troubleshooting

| Symptom               | Likely Cause                 | Recommended Fix                            |
| --------------------- | ---------------------------- | ------------------------------------------ |
| Message not routed    | No matching Choice condition | Review DataWeave expressions               |
| Scatter-Gather fails  | One route throws exception   | Add error handling or use First Successful |
| Round-Robin imbalance | Different consumer configs   | Align route configurations                 |
| High memory usage     | Large aggregated payloads    | Stream or batch process data               |

---

## 🔟 Summary Table

| Router               | Purpose              | Pattern Type | Key Feature                |
| -------------------- | -------------------- | ------------ | -------------------------- |
| **Choice**           | Conditional routing  | Sequential   | First matching route       |
| **Scatter-Gather**   | Parallel aggregation | Fork-Join    | Concurrent execution       |
| **Round-Robin**      | Load balancing       | Sequential   | Even distribution          |
| **First Successful** | Failover             | Parallel     | Returns first success      |
| **Until Successful** | Retry                | Loop         | Automatic retries          |
| **Flow-Ref**         | Modular logic        | Subflow      | Reusable, composable flows |
| **All**              | Sequential execution | Aggregation  | Executes all routes        |

---

## 11️⃣ Visual Summary

```text
Incoming Message
       │
    Choice Router
 ┌─────┴─────┐
When type A   When type B
   │             │
Scatter-Gather   Flow Ref
┌────┬────┐
Flow1 Flow2 Flow3
   │      │
   └──────┘
Aggregated Response
```

---

## ✅ Key Takeaways

* **Choice Router** → Conditional routing (if-else logic)
* **Scatter-Gather** → Parallel execution + aggregation
* **Round-Robin** → In-app load balancing
* **Flow-Ref** → Modular orchestration and reusability
* **Combine Routers** → Build complex, fault-tolerant message orchestration
* **Monitor performance** → Tune concurrency, memory, and error handling

---

