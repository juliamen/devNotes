# 🧩 **MuleSoft VM Queues – Complete Guide**

## 1️⃣ **Overview**

The **VM (Virtual Machine) Queue** in MuleSoft is a lightweight, **in-memory message queue** used for **inter-flow communication** within the same Mule application or runtime.

It enables **asynchronous message processing**, **load leveling**, and **decoupling** between producer and consumer flows.

**Common benefits:**

* ✅ Asynchronous processing
* ⚖️ Load leveling between flows
* 🔗 Decoupling high-throughput producers from slower consumers
* 🔁 Ordered message delivery within the same Mule runtime

---

## 2️⃣ **What is a VM Queue?**

### 🧩 **Definition**

A **VM Queue** is a **named in-memory queue** that holds messages for **internal Mule app consumption only**.

Messages are:

* **Published** using `vm:publish` or `vm:queue`
* **Consumed** using `vm:listener` (asynchronous) or `vm:consume` (synchronous)

> 💡 Persistent queues can optionally be enabled to survive Mule runtime restarts.

---

## 3️⃣ **VM Queue vs Object Store vs JMS**

| **Feature**            | **VM Queue**                 | **Object Store**         | **JMS (e.g., ActiveMQ)** |
| ---------------------- | ---------------------------- | ------------------------ | ------------------------ |
| **Scope**              | Intra-application            | Key–value storage        | Inter-application/system |
| **Persistence**        | Optional (persistent queues) | Optional                 | Always persistent        |
| **Asynchronous**       | ✅ Yes                        | ❌ No                     | ✅ Yes                    |
| **Use Case**           | Flow decoupling              | State tracking / caching | Cross-app messaging      |
| **Overhead**           | Low (in-memory)              | Low                      | Higher (external broker) |
| **Retry / Durability** | Only with persistent VM      | N/A                      | Built-in                 |

---

## 4️⃣ **VM Queue Operations**

| **Operation**             | **Purpose**                              |
| ------------------------- | ---------------------------------------- |
| `vm:publish` / `vm:queue` | Push a message into a VM queue           |
| `vm:listener`             | Consume messages asynchronously          |
| `vm:consume`              | Poll a queue synchronously               |
| `queueName`               | Name of the queue (shared across flows)  |
| `numberOfConsumers`       | Controls parallel consumption            |
| `persistent`              | Enables disk persistence across restarts |

---

## 5️⃣ **When to Use VM Queues**

✅ **Recommended Scenarios:**

* Asynchronous processing (e.g., API receives request → queued → backend processes later)
* Load leveling to handle traffic spikes
* Decoupling logic between flows
* Maintaining message order within a single flow
* Retrying failed processes without blocking the main thread

---

## 6️⃣ **Common Use Cases**

| **Use Case**           | **Example**                                                        |
| ---------------------- | ------------------------------------------------------------------ |
| **Async processing**   | User uploads CSV → VM Queue → Batch Flow processes asynchronously  |
| **Load leveling**      | High-volume API requests → queued → processed by limited consumers |
| **Flow decoupling**    | API layer publishes → Process layer consumes                       |
| **Retry mechanism**    | Failed transaction → Retry Flow via VM Queue                       |
| **Internal event bus** | Flows within one app communicate via VM queues                     |

---

## 7️⃣ **Configuring VM Queues**

### 🧩 **Step 1 — Define a Queue**

```xml
<vm:config name="VM_Config" doc:name="VM Config">
    <vm:queue name="AsyncQueue" persistent="true" maxOutstandingMessages="1000"/>
</vm:config>
```

### 🧩 **Step 2 — Publish Messages**

```xml
<flow name="ProducerFlow">
    <http:listener config-ref="HTTP_Listener_Config" path="/submit"/>
    <vm:publish queueName="AsyncQueue" doc:name="Publish to VM Queue">
        <vm:payload><![CDATA[#[payload]]]></vm:payload>
    </vm:publish>
    <logger message="Message queued" level="INFO"/>
</flow>
```

### 🧩 **Step 3 — Consume Messages**

```xml
<flow name="ConsumerFlow">
    <vm:listener queueName="AsyncQueue" numberOfConsumers="2" doc:name="Consume from VM Queue"/>
    <logger message="Processing message: #[payload]" level="INFO"/>
    <!-- Add processing logic -->
</flow>
```

✅ **Notes:**

* `numberOfConsumers` controls concurrency.
* If `persistent="true"`, messages persist through restarts.
* VM Queues are **intra-application only** — use **JMS/AMQP** for cross-app communication.

---

## 8️⃣ **Example: Async CSV Processing**

### **Producer Flow**

```xml
<flow name="CSVProducer">
    <file:listener path="/inbound" />
    <foreach collection="#[payload.rows]">
        <vm:publish queueName="VMQueueCSV" />
    </foreach>
</flow>
```

### **Consumer Flow**

```xml
<flow name="CSVConsumer">
    <vm:listener queueName="VMQueueCSV" numberOfConsumers="5" />
    <logger message="Processing row #[payload]" />
    <db:insert ... /> <!-- Insert into database or Salesforce -->
</flow>
```

🎯 **Benefit:** API responds instantly while heavy logic runs asynchronously.

---

## 9️⃣ **VM Queue Properties**

| **Property**             | **Description**                | **Default / Example** |
| ------------------------ | ------------------------------ | --------------------- |
| `queueName`              | Name of the queue              | `"AsyncQueue"`        |
| `persistent`             | Store messages on disk         | `false`               |
| `numberOfConsumers`      | Number of parallel consumers   | `1`                   |
| `maxOutstandingMessages` | Max messages before throttling | `1000`                |
| `timeout`                | Wait time for `vm:consume`     | `0` (infinite)        |

---

## 🔟 **Best Practices**

✅ Clearly name queues shared by multiple flows.
✅ Use **persistent queues** for critical data.
✅ Tune **numberOfConsumers** to balance performance.
✅ Avoid VM Queues for **cross-application** messaging — use JMS instead.
✅ Implement dead-letter handling for failed messages.
✅ Monitor queue size to prevent memory overload.

---

## 11️⃣ **Advantages and Limitations**

| **Aspect**      | **Advantage**            | **Limitation**                  |
| --------------- | ------------------------ | ------------------------------- |
| **Speed**       | Fast, in-memory          | Slower if persistent            |
| **Persistence** | Optional durability      | Not distributed                 |
| **Scope**       | Decouples internal flows | Single app/runtime only         |
| **Complexity**  | Simple to use            | Limited monitoring              |
| **Throughput**  | High                     | Memory-bound with large volumes |

---

## 12️⃣ **Troubleshooting Tips**

| **Symptom**              | **Possible Cause**          | **Solution**                           |
| ------------------------ | --------------------------- | -------------------------------------- |
| Messages not consumed    | Queue name mismatch         | Verify producer & consumer queue names |
| High memory usage        | Too many in-memory messages | Use persistence or throttle            |
| Messages lost on restart | `persistent="false"`        | Enable persistence                     |
| Slow processing          | Single consumer             | Increase `numberOfConsumers`           |
| Flow blocked             | Busy consumer               | Make flow async or add threads         |

---

## 13️⃣ **Summary**

| **Concept**         | **Key Takeaway**                             |
| ------------------- | -------------------------------------------- |
| **Purpose**         | Lightweight intra-app message queue          |
| **Core Operations** | `vm:publish`, `vm:listener`, `vm:consume`    |
| **Persistence**     | Optional durability with `persistent="true"` |
| **Concurrency**     | Controlled via `numberOfConsumers`           |
| **Limitations**     | Single-app only, memory bound                |

---

## 14️⃣ **Diagram: VM Queue Flow**

```
 ┌─────────────┐
 │ Producer    │
 │ Flow/API    │
 └─────┬───────┘
       │ vm:publish
       ▼
 ┌─────────────┐
 │  VM Queue   │
 │ AsyncQueue  │
 └─────┬───────┘
       │ vm:listener
       ▼
 ┌─────────────┐
 │ Consumer    │
 │ Flow/Logic  │
 └─────────────┘
```

