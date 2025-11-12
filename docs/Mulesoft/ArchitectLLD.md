
# Mulesoft Low Level Design (LLD)

## Deployment model

* **Environments:** dev -> test -> pre-prod -> prod
* **Runtime:** MuleSoft Private Cloud/Runtime Fabric or CloudHub depending on infra
* **Instances:** Use cluster for high availability
* **Secrets:** Use Anypoint Secrets Manager or Vault integration

## Mule applications and artifact layout

1. `acme-exp-sales-inbound` (Experience App)

   * HTTP Listener: `/exp/sales/v1/record-change`
   * Policies applied in API Gateway: rate-limit, IP allowlist, client-id enforcement
   * Flow steps: Auth filter -> Validate payload -> enrich (lookup) -> forward to `acme-proc-sales-to-3p`

2. `acme-proc-sales-to-3p` (Process App)

   * Receives internal REST call
   * Steps: validate -> dedupe (Redis/Cache or DB lookup) -> DataWeave transform -> call `acme-sapi-thirdparty-rest` -> process response -> audit log

3. `acme-sapi-thirdparty-rest` (System App)

   * Handles HTTP Request to ThirdParty API
   * Responsibilities: auth token refresh, timeout & retry policies, circuit-breaker, transform errors to canonical error schema

4. `acme-sched-sql-extract` (Scheduler App)

   * Uses Scheduler (Quartz / Mule scheduler) with cron expression
   * Calls `acme-sapi-sql-db` to fetch new/changed rows since last successful timestamp (stored in state store)
   * Saves last-run checkpoint atomically

5. `acme-sapi-sql-db` (System App)

   * JDBC connector with parameterized queries and paging
   * Exposes REST endpoint consumed by Scheduler

6. `acme-proc-sql-to-sales` (Process App)

   * Transforms rows into Salesforce composite upsert format in batches (size 200 or composite limits)
   * Calls `acme-sapi-salesforce`

7. `acme-sapi-salesforce` (System App)

   * Uses Salesforce Connector with OAuth2 refresh; handles upsert and composite
   * Handles rate-limit and bulk options if volume is large

## DataWeave mapping examples

**Simple SalesToThirdParty transform (DW 2.0)**

```dw
%dw 2.0
output application/json
var record = payload.record
---
{
  clientId: record.Id,
  companyName: record.Name,
  address: {
    line1: record.BillingAddress.street default "",
    city: record.BillingAddress.city default "",
    postcode: record.BillingAddress.postalCode default ""
  }
}
```

**SQL row to Salesforce sObject batch**

```dw
%dw 2.0
output application/json
var rows = payload.rows
---
rows map ((r, idx) -> {
  attributes: { type: "Contact" },
  External_Id__c: r.external_id,
  Name: r.name,
  Email: r.email
})
```

## Error payload (canonical)

```json
{
  "errorId": "uuid",
  "correlationId": "ACME-SALES-1234",
  "api": "acme-proc-sales-to-3p",
  "status": 502,
  "message": "Downstream third-party returned 502",
  "details": { }
}
```

## Transactional behavior, idempotency & dedupe

* For inbound Salesforce updates, the Flow should include an `eventId` or correlation id. The `proc` layer checks a dedupe store (Redis or DB table with TTL) before processing.
* For scheduler: use checkpointing of `lastProcessedTimestamp` in a durable store (database or Redis). If a run fails, the scheduler retries using the previous checkpoint.

## Retry policy / Circuit breaker

* Retries: 3 attempts with exponential backoff (500ms, 2s, 8s).
* Circuit breaker: open if failure rate > 50% over 1 minute; half-open test every 30s.

## Monitoring & Dashboards

* Dashboards per API: throughput, p95 latency, error rate, downstream call success.
* Logs: capture request and response sizes, truncated bodies for privacy.
* Business metric events: number of records processed per run, number of failed records.

---

## Operational runbooks (summary)

* **On failed downstream (3rd party)**: check SAPI-Third logs, rotate token if auth failures, re-run message from DLQ, escalate if > 5 failures.
* **On partial Salesforce upsert failures**: extract failed records to a CSV, attempt idempotent retry, or manual remediation via UI.
* **On scheduler failure**: restart app, examine last checkpoint; if missing, run range query to reprocess.

---

## Testing strategy

* Unit tests: DataWeave transformations using MUnit.
* Integration tests: mock third-party using Wiremock; test partial failures.
* Performance test: load test scheduler upsert volumes and ensure SLA.
* Security test: penetration test on APIs and validate secrets handling.

---

## Artifacts to deliver

1. RAML / OAS for all APIs (Experience, Process, System)
2. Mule application source (Maven project layout) with MUnit tests
3. DataWeave transformation specs and sample payloads
4. Runbooks and rollout checklist
5. Prometheus/Anypoint dashboards and alert rules

---

## Next steps / decisions needed from stakeholders

* Confirm expected message volumes and peak throughput to size concurrency and batching.
* Confirm authentication method Salesforce will use (Platform Event vs HTTP -> Named Credential / JWT).
* Confirm whether third-party supports bulk/batch endpoints to optimise throughput.
* Confirm SLA and business error handling preferences (synchronous error to Salesforce or asynchronous acknowledgement).

---

