# Groundwater Monitoring Data Store — Project Plan

## 1. Goal

Build a self-hosted system that receives water-level telemetry forwarded by Processr.IO, preserves the original data locally, and provides dependable viewing, review, and export without relying on the Processr web interface.

The initial release should:

- receive new readings automatically;
- retain both the unmodified inbound message and normalized measurements;
- make device history, current state, and data gaps easy to inspect;
- run under the existing Coolify instance;
- avoid exposing PostgreSQL or opening a new application-specific firewall port; and
- be recoverable from backup.

## 2. What is known and what must be confirmed

Processr publicly describes support for HTTP connectivity, but its public site does not document authentication, retry policy, or historical export capability. A representative `POST` payload has now been supplied. Processr sends one request for each message at a timestamp; related messages form a transaction:

```json
{
  "trx": {
    "id": 123,
    "received": "2026-09-17T23:25:26.168Z",
    "count": 2
  },
  "device": {
    "id": 2,
    "address": "ABC123",
    "altId": "AlternativeIdValue",
    "attributes": { "user_attr": "value" },
    "properties": [
      {
        "name": "floatProp",
        "attributes": { "user_prop_attr": "value" }
      }
    ]
  },
  "metadata": {
    "index": 0,
    "isEvent": true,
    "timestamp": 1789687526168,
    "time": "2026-09-17T23:25:26.168Z"
  },
  "data": {
    "floatProp": 2.12,
    "boolProp": false,
    "numberProp": 123
  }
}
```

This establishes that:

- Processr can send either `POST` or `PUT`; this project will use `POST` because each delivery adds a message rather than replacing a resource identified by the request URL;
- the request is configured as JSON using `Content-Type: application/json`;
- custom request headers can be configured, allowing a secret to be sent outside the URL;
- a request identifies the transaction, device, message position, and expected transaction message count;
- timestamps are supplied as both ISO-8601 UTC and Unix epoch milliseconds;
- one message can contain several dynamically named values of different scalar types;
- device- and property-level attributes are extensible; and
- transaction completion can be checked by receiving every index from `0` through `trx.count - 1`.

Still to confirm through the account interface, vendor documentation, or a controlled test:

1. Is the transmitted body strict JSON (without the explanatory comments shown in the example)?
2. Does Processr require publicly trusted HTTPS?
3. What response code/body and response time indicate success?
4. Does it retry on timeout, network failure, `429`, or `5xx`; for how long?
5. Is `trx.id` globally unique, account-scoped, device-scoped, or eventually reused?
6. Can messages within a transaction arrive out of order, and can two transactions interleave?
7. Do all messages in a transaction share the same device, timestamp, `trx.received`, and `trx.count`?
8. What does `metadata.isEvent` mean operationally?
9. What scalar value types can appear in `data` (`number`, Boolean, string, or null), and can objects/arrays occur?
10. Where are engineering units, quality/status, battery, signal strength, and device/property names configured or transmitted?
11. Is `device.properties` a complete channel list or only metadata for selected properties?
12. Can old readings be replayed or exported for backfill?
13. Is a pull API available if forwarding cannot reach the local system?
14. Are there documented source IP addresses or request-signing options?

The test should capture request method, URL/query fields, selected headers, exact raw body, and Processr's handling of `2xx`, `4xx`, timeout, and temporary `5xx` responses. Use a test device or non-sensitive sample data.

## 3. Recommended first-release architecture

```text
Processr.IO cloud
        |
        | HTTPS POST (preferred)
        v
Coolify proxy / TLS
        |
        v
Ingestion API --------------> PostgreSQL
  - authenticate                 - raw_ingest_events (JSON/text)
  - validate                     - devices and sites
  - deduplicate                  - measurement_channels
  - preserve raw input           - readings
  - normalize                    - review flags/notes
        |
        v
 health, logs and delivery-gap monitoring

Grafana ---------------------> PostgreSQL (read-only account)
  - latest values
  - time-series charts
  - gaps and device health
  - tabular review / CSV export
```

### Suggested components

- **Ingestion/API service:** Python with FastAPI. It is small, strongly supported, easy to validate and test, and can later provide read/export endpoints if needed.
- **Database:** PostgreSQL. Start with ordinary partition-friendly tables; do not require TimescaleDB until data volume or query performance justifies it.
- **Database access:** SQLAlchemy plus Alembic migrations.
- **Review interface:** Grafana for the first release. It covers time-series plots, tables, filters, alerting, and CSV export without committing the project to a custom front end.
- **Deployment:** a Docker Compose application managed by Coolify, containing the API and PostgreSQL. Grafana may be in the same stack or an existing shared Grafana instance.

PostgreSQL and Grafana's database credentials stay on a private Docker network. Only the API route and, if desired, the authenticated Grafana route are publicly routed by Coolify. PostgreSQL must not have a public host-port mapping.

## 4. Network approach without new firewall rules

Use the first viable option:

### Option A — Existing Coolify HTTPS ingress (preferred)

If another Coolify-hosted application is already publicly reachable through HTTPS, ports 80/443 and the proxy path are probably already available. Add a dedicated hostname such as `gw-ingest.example.org` and route it through the existing Coolify proxy to the API container. This adds no new port or direct database exposure.

This assumption must be verified from outside the organisation's network. A reachable Coolify dashboard alone is not proof that arbitrary application domains are routed correctly.

### Option B — Outbound-only tunnel

If the server has no usable inbound HTTPS route and policy permits it, use a Cloudflare Tunnel or an equivalent approved service. The connector initiates outbound connections, so no inbound port-forward or public origin IP is required. This requires authority to manage the chosen domain/tunnel and should be reviewed against organisational security policy. It also depends on the network already permitting the tunnel's outbound connection; Cloudflare Tunnel normally uses outbound TCP or UDP port 7844.

Protect the tunnel hostname with machine-to-machine authentication at the edge, not only in the application. Cloudflare Access service tokens normally use `CF-Access-Client-Id` and `CF-Access-Client-Secret` headers. If Processr permits only one additional custom header, Cloudflare Access can instead be configured to read both service-token values from one chosen header. Where Processr permits enough custom headers, retain a separate application webhook key for defence in depth.

This option prevents direct exposure of the Coolify server and API origin, but an Internet-addressable hostname still exists at the tunnel provider's edge. Requests without the service token are rejected there before they enter the tunnel.

### Option C — Pull or relay

If neither public ingress nor an approved tunnel is possible, determine whether Processr provides a read API, scheduled export, or vendor-hosted relay. A scheduled job on the Coolify server could then make outbound requests and import readings. This is only viable if Processr exposes such a facility; its public website does not establish that it does.

An HTTP forward cannot directly reach a private LAN address from the Processr cloud. Some form of existing public route, outbound tunnel, or pull mechanism is therefore required.

If the requirement is stricter—no Internet-addressable receiver at all—Processr push delivery cannot work unless Processr participates in the same private network. The viable alternatives are a local scheduled pull from a Processr API/export, a vendor-supported Processr Bridge/on-prem arrangement, or a cloud queue/relay that receives the POST and is polled outbound by the local server. The last alternative keeps the local system private but adds a third-party data store and another failure/reconciliation boundary.

## 5. Ingestion contract and reliability

Prefer a single endpoint such as:

```text
POST /v1/processr/readings
Content-Type: application/json
X-Processr-Webhook-Key: <long random secret>
```

Configure a second custom header named `X-Processr-Webhook-Key` containing a randomly generated secret of at least 32 bytes. Store the matching value as a Coolify secret. A dedicated header is recommended over a query-string or path secret because URLs are commonly logged. If Processr later documents signed requests, HMAC signing with a timestamp and replay window would be a stronger future option.

Secret rotation can be performed without interrupting ingestion by allowing a current and next key for a short overlap, changing the Processr header, confirming successful delivery, and then removing the old key.

The endpoint should:

- accept only the configured method and content type;
- enforce a small request-size limit;
- authenticate before parsing expensive content;
- record receive time and a safe subset of transport metadata;
- store the exact raw body before or alongside normalization;
- validate known fields without discarding unknown fields;
- store all timestamps in UTC while retaining the original timestamp text;
- deduplicate messages using `(source, device.id, trx.id, metadata.index)` until the documented scope of `trx.id` is known;
- treat an identical retry as success, but record and alert if the same identity arrives with a different payload hash;
- accept out-of-order message indexes and track whether all `trx.count` messages have arrived;
- reject impossible indexes, non-positive counts, or inconsistent transaction metadata without overwriting accepted data;
- split each key/value pair in `data` into a typed observation linked to the source message;
- compare `metadata.timestamp` and `metadata.time`, preserve both, and flag a discrepancy rather than silently choosing between them;
- return a success response quickly after a durable database commit;
- return `4xx` for unauthenticated or permanently invalid requests and `5xx` for temporary storage failure; and
- never log secrets or full sensitive payloads by default.

For the expected modest device count, a transactional database write is simpler and safer than introducing a message broker. Add a queue only if testing shows burst volume or Processr response-time requirements make direct durable writes unsuitable.

## 6. Initial data model

Keep provider-specific input separate from the long-lived domain model.

### `sites`

- internal ID and human-readable name/code;
- optional coordinates, datum, ground/reference elevation, and timezone;
- active dates and descriptive metadata.

### `devices`

- internal ID, Processr/device identifier, serial/name, site, and active dates;
- model and provider metadata in `jsonb`;
- last-received timestamp.

### `measurement_channels`

- device, provider channel/tag, normalized quantity, and canonical unit;
- expected value type and latest provider property attributes in `jsonb`;
- optional scale/offset and expected reporting interval.

### `readings`

- channel and measurement timestamp;
- typed source value using nullable numeric, Boolean, text, and `jsonb` columns with a constraint that exactly one representation is selected;
- normalized numeric value and canonical unit where the channel represents a measurable quantity;
- original value/unit;
- provider quality/status;
- receive timestamp and source raw-event ID;
- uniqueness constraint for idempotency.

Each entry in the message's `data` object creates one reading. For the example, `floatProp`, `boolProp`, and `numberProp` become three readings sharing the same device, timestamp, transaction, and raw-message reference.

### `provider_transactions`

- source, provider transaction ID, device, provider receive time, and expected message count;
- first/last local receive times, received-index count, completion time, and status;
- uniqueness key scoped conservatively by source and device until Processr confirms the scope of `trx.id`;
- constraint/processing checks that later messages agree on transaction-level metadata.

### `raw_ingest_events`

- local receive timestamp, Processr transaction ID, message index, content type, raw body or `jsonb` payload;
- provider `trx.received`, `metadata.timestamp`, `metadata.time`, and `metadata.isEvent` values;
- payload hash, processing status, validation error, and processing version.

### `reading_flags` / `review_notes`

- reading or time-range reference;
- flag type, note, author, and timestamps;
- preserve original observations rather than editing them in place.

Retain `device.attributes`, `device.properties`, and unknown fields in the raw event. Update provider metadata snapshots without using those mutable attributes as reading identity. Do not lock the remaining columns until representative payloads from every device/channel type have been captured.

## 7. Viewing and review scope

The first Grafana dashboards should provide:

- latest reading and last-contact time for every device;
- water level versus time, filterable by site/device and date range;
- battery, signal, and diagnostic channels where available;
- missing-data gaps based on the expected reporting interval;
- suspect/invalid quality flags;
- a table suitable for review and CSV download; and
- an ingestion-health view showing accepted, duplicate, and rejected events.

Grafana should use a read-only PostgreSQL role. Editing flags or metadata can initially be performed through a small authenticated API/admin workflow. Build a custom application only when requirements such as annotations, approval states, corrections, reporting, or specialised groundwater calculations exceed Grafana's strengths.

## 8. Delivery phases

### Phase 0 — Discovery and proof of connectivity

Deliverables:

- completed Processr capability checklist;
- one or more captured sample requests with secrets removed;
- chosen networking option;
- evidence that Processr can reach a test HTTPS endpoint;
- documented retry/duplicate behaviour; and
- sample historical export, if available.

Exit criterion: a real test reading reaches an approved endpoint and the required fields and delivery semantics are understood.

### Phase 1 — Ingestion vertical slice

Deliverables:

- repository structure, Dockerfiles, Compose configuration, and environment template;
- FastAPI health and ingestion endpoints;
- migrations for raw events and the minimum normalized schema;
- authentication, validation, UTC conversion, and duplicate protection;
- automated unit and integration tests using captured, anonymised fixtures; and
- Coolify staging deployment.

Exit criterion: repeated delivery of the same sample results in one normalized reading, with every attempt traceable and no secret written to logs.

### Phase 2 — Production storage and backfill

Deliverables:

- mapping for all known devices and channels;
- production Processr forward configuration;
- historical import tool if export data is obtainable;
- operational logs and ingestion metrics; and
- reconciliation report comparing Processr and local counts over a test period.

Exit criterion: at least one complete reporting interval is reconciled with no unexplained loss or duplication.

### Phase 3 — Review interface

Deliverables:

- provisioned read-only Grafana data source;
- overview, time-series, data-quality, and ingestion-health dashboards;
- filters and CSV export; and
- access control for intended users.

Exit criterion: a user can find a site, inspect a chosen period, identify gaps/flags, and export its readings without using Processr.

### Phase 4 — Operations and recovery

Deliverables:

- automated database backups to storage independent of the server PC;
- retention policy and encrypted credential handling;
- a tested restore procedure;
- monitoring for failed ingestion, low disk space, backup failure, and devices overdue to report;
- update and secret-rotation procedure; and
- concise operator runbook.

Exit criterion: a clean database can be restored from backup and a simulated delivery outage is detected.

## 9. Testing strategy

- **Contract tests:** one anonymised fixture per observed Processr payload variant.
- **Parser tests:** missing fields, unknown fields, malformed numbers, units, timestamps, and quality values.
- **Idempotency tests:** exact retries and semantically duplicate readings.
- **Transaction tests:** indexes arriving out of order, interleaved transactions, a missing index, duplicate indexes, inconsistent counts, and the same identity with a changed payload.
- **Timestamp tests:** ISO/epoch agreement, mismatch handling, millisecond precision, and UTC storage.
- **Dynamic data tests:** numeric, Boolean, text, null/unsupported values, unknown properties, and changing property attributes.
- **Security tests:** missing/wrong credentials, oversized bodies, unsupported content types, and secret redaction.
- **Database tests:** migrations from an empty database and constraints under concurrent duplicate delivery.
- **End-to-end test:** Processr test device to public endpoint to PostgreSQL to dashboard.
- **Failure tests:** database unavailable, API restart during delivery, delayed/out-of-order data, and temporary `5xx` response.
- **Restore test:** recreate the database from the most recent independent backup.

## 10. Security and operational baseline

- HTTPS only; no direct public PostgreSQL port.
- Store secrets in Coolify environment secrets, not the repository.
- Use separate database roles for migrations/API and Grafana read-only access.
- Restrict the ingestion route by authentication and rate/request-size limits; add source IP filtering only if Processr publishes stable addresses.
- Keep raw data immutable and record parser/mapping versions for auditability.
- Back up off-machine; a Docker volume on the same server is persistence, not disaster recovery.
- Apply container and dependency updates on a defined cadence.
- Define a reasonable data-retention period for request metadata and logs; retain measurements according to business requirements.
- Avoid collecting unnecessary personal or network-identifying data in request logs.

## 11. Main risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Processr forward options are too limited | Weak authentication or unusable payload | Run Phase 0 first; ask vendor for integration documentation; consider pull/relay/on-prem bridge |
| No approved inbound route | Processr cannot deliver | Reuse existing 443 route, obtain approval for outbound tunnel, or use pull/export |
| Undocumented retries cause duplicates | Inflated or conflicting readings | Idempotent writes and raw-event audit trail |
| Provider fields or units change | Incorrect normalized data | Preserve raw input, version mappings, alert on unknown fields |
| Timestamps lack offsets | Readings placed at wrong time | Confirm timezone per device; retain original text; normalize to UTC |
| Server PC or disk fails | Loss of local archive | Automated off-machine backups plus restore tests |
| Custom UI expands scope early | Slow delivery and maintenance burden | Use Grafana first; derive custom UI requirements from real usage |

## 12. Decisions to make after discovery

- Processr delivery/retry behaviour and success response contract.
- Scope and reuse behaviour of `trx.id`, plus transaction ordering guarantees.
- Supported `data` value types and the meaning of `metadata.isEvent`.
- Existing Coolify ingress versus approved outbound tunnel versus pull.
- Whether historical data can be backfilled and in which format.
- Canonical units and groundwater-specific reference levels/datum handling.
- Required retention period and expected data volume.
- Who may view data and who may add flags/notes.
- Backup destination and retention policy.
- Whether Grafana alone is sufficient for the first operational release.

## 13. Definition of success

The project is successful when every expected device reading is durably stored locally, duplicates and failures are visible, authorised users can review and export history without Processr, and the system can be restored after loss of the Coolify host.
