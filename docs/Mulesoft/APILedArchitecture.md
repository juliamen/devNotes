
# MuleSoft API-Led Connectivity Architecture Guide

## Overview

This document describes a recommended **API‑Led Connectivity architecture** in MuleSoft, including:

- Component placement (DB connector, Salesforce batch/upsert, Salesforce listener, scheduler, DataWeave transformations)  
- Cross-cutting concerns: error handling, idempotency, security, monitoring, deployment  

**Target audience:** MuleSoft architects, integration developers, DevOps engineers

---

## API Layers and Responsibilities

Follow MuleSoft's API‑led approach, split into three layers:

| Layer | Responsibility |
|-------|----------------|
| **System APIs** (System Layer) | Encapsulate access to source systems (DBs, SaaS, mainframes). Connectivity, protocol translation, CRUD, backend throttling |
| **Process APIs** (Orchestration Layer) | Business logic and orchestration between System and Experience APIs. Transformations, aggregation, idempotency, retries, batching |
| **Experience APIs** (Experience Layer) | Tailored APIs for consumers (mobile, web, Salesforce Flows). Presentation mapping, minor orchestration, security |

---

## Component Placement Recommendations

### DB Connector

- **Place:** System API (close to data source)  
- **Rationale:** Centralize data access; isolate SQL/schema from business logic  
- **Implementation Notes:**  
  - Use connection pool (`validationQuery`, `maxActive`, `minIdle`)  
  - Avoid exposing raw SQL to Experience APIs; provide coarse-grained methods  
  - Parameterize queries to prevent SQL injection  

### Salesforce Pub/Sub Listener (Platform Events / CDC)

- **Place:** System API or lightweight Process API  
- **Rationale:**  
  - Simple transformation → System API  
  - Complex business processing → Process API  
- **Implementation Notes:**  
  - Use Salesforce Connector for Platform Events / CDC  
  - Durable consumer: replay IDs or platform event replay  
  - High throughput: buffer via Object Store v2, JMS, Kafka  

### Scheduler

- **Place:** System API or dedicated scheduler service  
- **Rationale:** Operational concern; avoid putting scheduling in Experience APIs  
- **Implementation Notes:**  
  - Use Mule Scheduler (Mule 4) or external schedulers (Control-M, AWS EventBridge, K8s CronJob)  
  - Call secure System API endpoint  

### Batch Processing (Salesforce / Large Files)

- **Place:** Process API or dedicated Batch App  
- **Rationale:** Handles business logic + large payloads  
- **Implementation Notes:**  
  - Use Mule Batch Job module (`chunkSize`, Input → Process → OnComplete)  
  - Salesforce Bulk API v2 for large upserts  
  - Tune chunking and parallelism to respect Salesforce limits  

### Upsert Records to Salesforce

- **Place:** Process API (or System API if centralizing Salesforce access)  
- **Rationale:** Combines business logic and external system keys  
- **Implementation Notes:**  
  - Use upsert with External ID to avoid duplicates  
  - Bulk API for large volumes  
  - Handle partial failures; retry or DLQ  

### DataWeave Transformations

- **Place:** Primarily in Process APIs  
- **Rationale:** Canonical models and business mappings live here  
- **Implementation Notes:**  
  - Keep complex logic in reusable `.dwl` modules  
  - Test with MUnit and sample payloads  
  - Prefer small, composable transforms  

### Error Handling

- **Place:** Across all layers (global + flow-specific)  
- **Rationale:** Consistency and observability  
- **Implementation Notes:**  
  - `on-error-propagate` and `on-error-continue`  
  - Meaningful HTTP statuses for Experience APIs (400, 401, 429, 500)  
  - Push errors to monitoring systems (ELK, Datadog, NewRelic)  
  - Retry transient errors with exponential backoff  
  - Non-retriable errors → DLQ or database  

### Idempotency

- **Place:** Process APIs  
- **Rationale:** Deduplicate unique business transactions  
- **Implementation Patterns:**  
  - **Idempotency Key:** Consumer-supplied header, persist in Object Store v2 / Redis / DB  
  - **Deduplication via business keys:** Compute deterministic key (e.g., `tenantId|eventType|timestampBucket`)  
  - **Idempotent consumer:** Make downstream operations repeat-safe  
- **Implementation Notes:**  
  - Short-term store: Object Store v2, long-term: DB  
  - Atomic operations using transactions or optimistic locking  

### Cross-Cutting Concerns

| Concern | Recommendations |
|---------|----------------|
| Security | Anypoint API Manager policies (Client ID/Secret, OAuth2, IP whitelisting, rate limiting); store credentials in secure properties; TLS for all comms |
| Logging & Tracing | Correlation ID across layers; log request entry, transformations, external calls, responses, errors; integrate with OpenTelemetry |
| Monitoring & Alerts | Request count, latency, errors, connector failures, queue lengths; alert on error spikes, retries, backlog |
| Retries & Circuit Breaker | Retry with exponential backoff; circuit breaker using `on-error-continue` and Object Store counters |
| Dead Letter Queue (DLQ) | Capture failed messages (SQS, Kafka, DB) including metadata and error details |

---

## Typical Flow Examples

### Example 1 — Scheduled File Processing & Salesforce Upsert

1. Scheduler triggers Process API (or dedicated batch app)  
2. Process API calls System API (FileSystemAPI / SFTPSystemAPI)  
3. Batch Job:  
   - Input: split file into records  
   - Process: DataWeave transform → canonical model → idempotency check  
   - Success: SalesforceSystemAPI.upsertContact  
   - Failure: DLQ  
   - OnComplete: summary notification + persist metrics  

### Example 2 — Real-Time Platform Event Listener

1. SalesforceSystemAPI subscribes to Platform Event / CDC topic  
2. On event receive: push envelope to internal queue (Object Store / Kafka) and ACK Salesforce  
3. Worker Process API consumes messages, validates, transforms, calls other System APIs  
4. Idempotency via event replay ID or unique event key  

---

## Sample Mule XML Snippets

**Global DB Config**

```xml
<db:postgresql-config name="Postgres_Config" host="${db.host}" port="${db.port}" user="${db.user}" password="${db.password}" database="${db.name}" doc:name="Postgres Config"/>
````

**Scheduler Flow**

```xml
<flow name="scheduled-file-flow">
  <scheduler frequency="3600" timeUnit="SECONDS"/>
  <flow-ref name="Process.File" />
</flow>
```

**Global Error Handler**

```xml
<error-handler>
  <on-error-propagate logException="true" enableNotifications="true" type="ANY">
    <set-payload value="#[{error: error.description, correlationId: vars.correlationId}]" />
    <logger level="ERROR" message="Global error: #[error.description]" />
  </on-error-propagate>
</error-handler>
```

**Idempotency Check (Pseudo DataWeave)**

```dw
%dw 2.0
output application/java
---
var idKey = attributes.headers['Idempotency-Key'] default (payload.id as String)
if (objectStore.get(idKey) != null) do
  // return stored response
else do
  objectStore.put(idKey, {response: payload, createdAt: now()}, ttl=86400)
  // continue processing
```

---

## Testing & QA

* **Unit tests:** MUnit for flows & DataWeave transforms
* **Integration tests:** Mock HTTP/SOAP/Salesforce sandbox endpoints
* **Load tests:** Emulate parallelism & volume; tune connection pools and bulk sizes

---

## Deployment & CI/CD

* Package each API as a separate Mule application
* Use CI/CD (Anypoint, GitHub Actions, Jenkins) for build, MUnit, staging, and prod deployment
* Separate environments: Dev / QA / Prod with environment-specific secure properties

---

## Operational Runbook (Short)

* Restart listener flows
* Replay failed platform events (if persisted)
* Reprocess DLQ items
* Inspect Object Store entries for idempotency

---

## Tradeoffs & Considerations

* Centralized System APIs → easier maintenance, adds extra hop/latency
* Transformations: Prefer Process APIs for canonical mapping
* Batch vs Real-Time: Bulk API cheaper for large Salesforce writes but adds latency

---

## Appendix: Checklist for Implementing an API

* Create API spec (RAML or OAS) → publish to Exchange
* Implement System API per backend
* Implement Process API for orchestration & transformations
* Add Experience APIs per consumer
* Add CI/CD pipelines & environment configs
* Implement idempotency, DLQ, retries, monitoring

