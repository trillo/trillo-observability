# Telemetry Ingestion Endpoint — Design

**Status:** Design (v0.1). **Priority:** P1 (observability integration; demo-useful).
**Context:** a lightweight path to ingest real-agent telemetry **into Postgres**
for small setups where OliverDB is overkill, landing in the *same* observability
schema so all downstream analytics (SLOs, drift, alerts, dashboards) are unchanged.

---

## 1. Purpose & scope

- Let a **real/external agent** send OpenTelemetry to Trillo AOS Observability with
  an **app-specific credential**, routed to the correct application, and have it
  flow into the platform for monitoring, SLOs, drift, alerts, and analytics.
- **Small-setup / demo path.** Same downstream as OliverDB — it writes into the
  observability app's existing `otlp_span / otlp_log / otlp_event / otlp_metric`
  tables, so sweepers and every screen keep working with no changes.
- **Non-goals (v1):** production-scale throughput (that's OliverDB), payload-level
  integrity signing (HMAC), and a custom non-OTLP wire format.

## 2. Decisions

- **AD-1 — OTLP-native only.** Implement the OTLP/HTTP receiver contract; no custom
  batch format. Standard OTel SDKs work with only endpoint + header config.
- **AD-2 — Dedicated `IngestionKey` class in `schema0`** (platform level), not the
  app-config secret store. Rationale: the write path resolves `key → appId` before
  it knows the app (must be app-agnostic, like `hosted_app_tbl` / `oauth_*`), and
  the key carries lifecycle + metering that a secret store shouldn't.

## 3. Endpoint contract (OTLP/HTTP)

Base endpoint (per environment), e.g.:
`https://aos-dev.trillo.ai/api/v2.0/telemetry`

The OTel exporter appends the signal path, so we serve:
- `POST …/telemetry/v1/traces`
- `POST …/telemetry/v1/logs`
- `POST …/telemetry/v1/metrics`

- **Encoding:** `application/x-protobuf` (exporter default) — accept protobuf;
  JSON (`application/json`) optional/nice-to-have.
- **Body:** OTLP `Export{Trace,Logs,Metrics}ServiceRequest`.
- **Response:** OTLP `Export…ServiceResponse` (200; empty/partial-success OK for v1).
- **New dependency:** OTLP protobuf decoding on the Java side (`opentelemetry-proto`).

**Agent side = config, not code.** The agent runs a standard OTel exporter:
```
OTEL_EXPORTER_OTLP_ENDPOINT = https://aos-dev.trillo.ai/api/v2.0/telemetry
OTEL_EXPORTER_OTLP_HEADERS  = x-trillo-ingest-key=<KEY>     # or Authorization=Bearer <KEY>
```
Exports are **batched and sent in the background**, so ingestion adds **no latency**
to the agent's own request path.

## 4. Auth & routing — `IngestionKey`

`schema0.ingestion_key_tbl` (platform):

| Field | Purpose |
| :-- | :-- |
| `id` | PK |
| `key_hash` | SHA-256 of the key; **plaintext shown once at creation** |
| `key_prefix` | first ~8 chars, for display/identification |
| `app_id` | target application (→ schema) |
| `label` | human name ("prod fleet exporter") |
| `status` | active / revoked |
| `created_by`, `created_at`, `revoked_at` | lifecycle |
| `request_count`, `bytes_ingested`, `last_used_at` | **metering** (see §6) |

**Validation flow (per request):**
1. Read the key header → look up `key_hash` in `schema0.ingestion_key_tbl`
   (cached, sub-ms). Reject if missing/revoked → `401`.
2. Resolve `app_id` → switch into that app's schema (same `setAppId` +
   `ConnectionHolder` pattern used by the OAuth login resolution).
3. Decode + write the OTLP batch into that app's `otlp_*` tables.

- **Owner-managed** in the AOS client (mint / label / revoke / rotate). Shared
  "owner manages credentials" surface with AOS-11 — **different backing** (AOS-11 =
  config secrets the app *reads*; this = a credential others present to *write*).

## 5. Data landing

- Decode OTLP → map to `otlp_span / otlp_log / otlp_event / otlp_metric` rows in
  the app schema. Reuse the mapping semantics from `OliverDB-Otel-Mapping-Requirements.md`,
  implemented **server-side in AOS** (Java) for the Postgres path.
- Batched `INSERT`; preserve `trace_id / span_id / execution_id` correlation and
  the `gen_ai.*` → column mapping (tokens, cost, model, tool).
- Sweepers/rollups then run unchanged — this endpoint only *fills the tables*.

## 6. Metering (the reason for a dedicated table)

- Per-key: `request_count`, `bytes_ingested`, `last_used_at` (and optionally per-day
  rollups for quota/billing).
- **Do not hot-update the key row per request** (write amplification). Buffer
  counters in memory and **flush periodically** (timer or a small sweeper);
  `last_used_at` at coarse granularity is fine.
- Enables per-app/per-key **quotas + rate limits** and a usage view for the owner.

## 7. Security

- TLS; keys **hashed at rest** (never stored/logged in plaintext; shown once).
- Per-key **rate limit** + **payload size cap**; reject oversized/malformed batches.
- Key `revoke` is immediate (cache TTL short, or invalidate on revoke).
- v1 omits HMAC payload signing — Bearer key + TLS is sufficient; revisit per demand.

## 8. Relationship to OliverDB

Per app (or per env), telemetry lands via **either** OliverDB **or** this Postgres
endpoint — the same `otlp_*` schema either way. A per-app switch (AppConfig flag or
deployment choice) selects the active ingestion path; downstream is identical.

## 9. Sequencing / dependencies

- **Do the REST authz fix (AOS-08/09) first** — this endpoint's auth builds on the
  same request-authorization layer, and a hardened base avoids repeating the gap.
- Build order: `IngestionKey` class + owner UI → OTLP receiver (traces first) →
  mapping to `otlp_*` → metering flush → rate limits → logs/metrics signals.

## 10. Open items

- Metering flush mechanism (in-process timer vs a `SweeperRun`-style job).
- `opentelemetry-proto` Java lib choice + version; JSON encoding in v1 or later.
- Default rate-limit / payload caps.
- Whether the OTel **metrics** signal is needed in v1 (many agent setups emit
  token/cost as span attributes, already covered by traces).
- Where the owner UI lives (AOS client) and how it overlaps AOS-11's credential UX.
