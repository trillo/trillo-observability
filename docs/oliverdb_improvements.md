# OliverDB — Suggested Improvements for AI Observability

**Document Version:** 0.1 (draft)
**Status:** For discussion with the **OliverDB team** — a running wishlist framed by the Trillo AI-agent observability workload.
**Companion to:**
- `OliverDB-Otel-Mapping-Requirements.md` (ingest-side mapping, in-process Rust plugin option)
- `Telemetry-Ingestion-Endpoint-Design.md` (Postgres small-setup ingestion contract)
- `Enterprise_AI_Agent_Observability_Gap_Analysis_and_OliverDB_Interface.md`

The OliverDB team has confirmed 3M events/sec ingest capacity on their Apache Arrow fork. Everything below is **functional / API-surface** work that unlocks specific query patterns and integration flows for AI-agent observability — the raw throughput and storage engine are not in question.

Items are grouped by category and tagged **P0 / P1 / P2** by their commercial-and-technical value to Trillo Agent Observability, not by OliverDB's own priorities.

---

## 1. Ingest surface

### 1.1  Native OTLP ingest endpoint  — **P0**

**What.** Add `POST /v1/traces` (and eventually `/v1/logs`, `/v1/metrics`) accepting standard **OTLP/HTTP** payloads (JSON + protobuf) alongside today's positional-cells `/v1/ingest`. Storage stays the frozen-column layout; the endpoint is a wire-shape adapter.

**Why for AI observability.**
- Every OpenTelemetry SDK — Google ADK, LangChain, CrewAI, LlamaIndex, raw OpenAI/Anthropic/Vertex — already emits OTLP. A native endpoint means **zero custom exporter code** to onboard a customer's fleet.
- Removes a translation layer on the producer side; removes an on-call surface (a translator would be another thing to keep healthy).
- Lets us drop OliverDB in as an OTLP destination in front of any collector (OpenTelemetry Collector, Vector, Grafana Agent, Alloy).

**Suggested behavior.**
- HTTP + gzip (protobuf-native and JSON-native), and optionally gRPC.
- OTLP v1.4+; accept resource + scope + span/log/metric hierarchies verbatim.
- Success semantics match today: default async `202 {"buffered": n}`, `?sync=1` for `200 {"ingested": n}`.
- Errors: return standard OTLP partial-success payload for well-formed batches with some rejected rows.

### 1.2  Semantic-convention pull-outs on ingest  — **P0**

**What.** On ingest, automatically extract these OTel semantic-convention keys from `attributes` into first-class columns:

| OTel semconv key | OliverDB column |
|---|---|
| `gen_ai.usage.input_tokens` | `input_tokens` |
| `gen_ai.usage.output_tokens` | `output_tokens` |
| `gen_ai.request.model` | *(still in attrs, but pre-indexed)* |
| `gen_ai.system` | *(still in attrs, but pre-indexed)* |
| `gen_ai.response.finish_reasons` | *(still in attrs, but pre-indexed)* |

**Why for AI observability.**
- Token accounting has to be **summable**. Attrs are string-typed / non-summable in OliverDB. Currently every producer has to know to promote these into cells; that's brittle across three-plus agent frameworks.
- Pre-indexing `gen_ai.request.model` / `gen_ai.system` massively speeds "spend by model" and "traffic by provider" dashboards.

**Suggested behavior.**
- Deterministic mapping table, versioned. Log a one-time WARN if the same span carries both a top-level `input_tokens` cell and a semconv key — reject only if they disagree.
- Configurable per key: promote to column, promote to indexed attribute, or leave in blob.

### 1.3  UPSERT on `(trace_id, span_id)` for late-arriving span data  — **P1**

**What.** Support `INSERT … ON CONFLICT (trace_id, span_id) DO UPDATE` at ingest.

**Why for AI observability.**
- OpenTelemetry emitters commonly write a span in two phases: **`spanStart`** at the beginning, **`spanEnd`** with `duration_us`, `status_code`, and final attributes when the operation finishes. Without UPSERT, exporters must buffer half-spans in memory until end — adds latency and memory pressure and loses spans on crash.
- Streaming LLM calls in particular emit partial-completion events over 10-60s; a native late-arrival path matches the shape of the source data.

**Suggested behavior.**
- If neither side has status_code, later write wins per field.
- If both sides supply a summable field (tokens), take the later value (final is authoritative). Alternatively, expose an `ingest_mode: last-write-wins|greatest|sum` per column.

### 1.4  A `Text` column + `text_match()` support in the observability schema  — **P1**

**What.** The current schema has no `Text` column. Add one — call it `body` — indexed for `text_match()`, carrying the OTLP LLM span's completion (`gen_ai.completion`) verbatim. The corresponding input (`gen_ai.prompt`) stays in `attrs` — `attrs.gen_ai.prompt` LIKE-prefix queries are index-accelerated for lookup, and the completion carries most of the analytical value anyway (hallucination triage, drift, output-shape analysis).

**Why single, not paired.** OliverDB's current schema constraint (per `oliverdb_onboarding_admin.md`, verified 2026-08-26): **"At most ONE Text column per schema — the FTS sidecar indexes a single text field; a second is rejected at `POST /v1/schema`."** So an earlier draft of this section calling for `input_text` + `output_text` was wrong. Recommendation is now one column.

**Why for AI observability.**
- "What did the agent actually say?" is the single most common ad-hoc query in incident response. Today's `attrs.gen_ai.completion` LIKE pattern is not indexed for full-text and is length-bounded by the attrs blob.
- Enables hallucination triage, output-shape drift analyses, and grounding-based verification.

**Suggested behavior.**
- One dedicated `Text` column on the span row: `body`. Truncation limit disclosed and configurable.
- `text_match()` with a minimal query grammar (AND / OR / phrase / prefix).

**Longer-term ask (P2, separate item):** if OliverDB team can add support for **N Text columns per schema** (each backed by its own FTS sidecar), we can split back into input + output. That's a real product ask worth discussing separately — the single-Text limit is friendly to storage but limits the observability queries we can build.

### 1.5  Streaming ingest via Kafka / PubSub  — **P2**

**What.** Accept a native Kafka topic or GCP PubSub subscription as an ingest source, not only HTTP.

**Why.**
- HTTP is fine for demos; production customers want durable buffering upstream (backpressure, replay, exactly-once at the boundary). Every large observability customer already runs Kafka.
- Removes a producer-side outage class (spike + HTTP retries).

### 1.6  gzip on the HTTP ingest path  — **P2**

**What.** Accept `Content-Encoding: gzip` on `/v1/ingest` and the OTLP endpoint.

**Why.** ~5-10× wire savings on JSON payloads; every OTel SDK sets this by default. Cheap; no algorithm change.

---

## 2. Query surface

### 2.1  Common Table Expressions (CTEs)  — **P1**

**What.** `WITH cte AS (…) SELECT …` support, including multiple CTEs.

**Why for AI observability.**
- Every non-trivial dashboard tile is a two-step query today: aggregate then filter or join. Without CTEs the app tier stitches results — costs a round-trip and forces "one query = one dashboard tile" limits.
- Multi-step aggregations (trace-level rollups then session rollups then per-service comparisons) become one query.

**Cost/effort read.** Parser-level feature. The executor already runs compiled plans; CTEs mostly desugar. Highest value-per-line-of-code addition on this list.

### 2.2  Broadcast JOIN against small right-hand tables  — **P1**

**What.** `SELECT … FROM t JOIN small_dim d ON t.k = d.k` where `small_dim` is < a few million rows (broadcast/dictionary-loaded to every worker).

**Why for AI observability.**
- **Cost calculation.** Join to a `model_pricing(model, $/1M_input, $/1M_output)` table to compute $ spend per query. Today the app tier keeps a pricing map and multiplies client-side — every change to prices requires an app-tier release.
- **Service metadata enrichment.** Join `service_name → owner_team, SLO_threshold, business_unit`. Without this the resource_attrs have to duplicate this info on every span.
- **Trace × business entity.** Join `attrs.session.id` to a session dimension to pivot latency by enterprise customer or subscription tier.

**Suggested behavior.**
- Only broadcast joins in v1 (right table replicated in RAM per worker). No shuffle-based joins.
- Explicit `HINT BROADCAST` optional; automatic when right-hand table is below a threshold.
- Right-hand tables managed via the same ingest path as `t`, but tagged as dimension.

### 2.3  Correlated subqueries — at least for aggregate-then-filter  — **P1**

**What.** `SELECT … FROM t WHERE ts IN (SELECT max(ts) FROM t GROUP BY trace_id)`.

**Why for AI observability.**
- **"Traces whose total duration > 30s and had ≥1 error span."** Inner sum-per-trace, outer filter.
- **"Latency for sessions that touched model X compared to sessions that didn't."** Cohort selection subquery.
- **"Spans slower than their service's p95 for the same span_name."** Correlated aggregate.

Without these the app tier does N+1 queries. On a 3M-events/sec store this is measurable dashboard latency, not just theoretical elegance.

### 2.4  Materialized view refresh policy  — **P2**

**What.** OliverDB already has "rollups". Expose them as declarative materialized views the customer can define, with named refresh cadence (`REFRESH EVERY '5 minutes'`) and optional incremental refresh.

**Why for AI observability.**
- Dashboards read a small number of aggregates over and over. Ad-hoc queries against raw spans are slower; against pre-aggregated views they're instant.
- Also unlocks alerting on rate-of-change of a rollup, not raw spans.

### 2.5  `PERCENTILE_CONT`, `PERCENTILE_DISC` on any column  — **P2**

**What.** Percentiles beyond the pre-built `latency` sketch — arbitrary column, arbitrary percentile, arbitrary GROUP BY.

**Why.** Token distributions, retry counts, embedding sizes — all deserve percentile analysis. The sketch is fast for the common case; a slower general path for the long tail is fine.

---

## 3. Retention, governance, and multi-tenancy

### 3.1  Retention policy per column-set / per-tenant  — **P1**

**What.** Beyond the global 3-day hot / 180-day archive, allow declarative retention such as:

```
RAW spans:            30 days
attrs.gen_ai.prompt:  7 days   (redacted after)
rollups (by_operation): 2 years
```

**Why.**
- Different customers, different compliance postures. GDPR + HIPAA + PCI-DSS all demand narrower windows on the sensitive columns than on the aggregates.
- Enables "keep the shape of the traffic long enough to trend, drop the payloads early."

### 3.2  DELETE by predicate  — **P0 for enterprise procurement**

**What.** `DELETE FROM t WHERE user_id = 'x' AND ts BETWEEN a AND b`.

**Why.**
- GDPR right-to-be-forgotten, CCPA equivalents, HIPAA correction requests.
- Without it, enterprise procurement is a hard block. Many analytical stores add this as a soft-delete / compaction pass — that's fine as long as reads honor it immediately.

### 3.3  Row-level scoping in scoped keys  — **P1** (partly done today per docs)

**What.** The `/onboard` doc says scoped keys can enforce row pins, column redaction, and window caps — expose the primitive to add **`WHERE resource_attrs.tenant_id = $1`** style row pins on the read-side of every query issued under that key.

**Why for AI observability.**
- A SRE Copilot agent given a scoped key should only ever see the tenant it was minted for, at the storage layer, not at the app layer. The engine rewrites out-of-scope queries — this doc item is about ensuring the primitive supports **attribute-key predicates**, not only frozen columns.

### 3.4  Column-level redaction in scoped keys  — **P2**

**What.** Scoped keys that null out `input_text` / `output_text` for consumers that don't have PII authorization.

**Why.** Analysts running latency queries never need prompt bodies. Redaction at the engine, not the SQL, is safer than "trust the query."

---

## 4. Schema additions & signal coverage

Today's OliverDB slot exposes a **single spans table**. The Trillo AI-agent observability emitter (see `generate_live_telemetry.py`) also produces two other OTel signals — **logs** and **events** — that have no home in OliverDB today. This section is our formal ask.

### 4.1  Add an OTLP-logs table  — **P0**

**What.** A second table (say `logs`, or a `signal_kind` discriminator on a unified table) shaped for OTLP log records.

Suggested column set (mirroring the OTLP `LogRecord` and the way spans are laid out today):

| Column | Type | Notes |
|---|---|---|
| `trace_id`, `span_id` | Id | correlates to the spans table (nullable — some logs are unattached) |
| `service_name`, `severity_text` | Keyword | filter / group |
| `severity_number` | Int | OTel numeric severity (1-24) |
| `body` | Text | log message body — `text_match()`-indexed |
| `resource_attrs`, `attrs` | Attrs | same long-tail model as spans |
| Implicit `ts` | epoch ms | log timestamp |

**Why for AI observability.**
- Every agent framework emits both spans **and** logs — LangChain callbacks, ADK's error records, tool-call stderr. Without a logs signal we lose the "what went wrong at 03:12" narrative and are stuck reconstructing incidents from span attributes.
- The current emitter writes `OtlpLog` rows to Postgres because there's nowhere else for them. With `oliverdb_improvements.md §4.1` landed, those go to OliverDB and the Postgres `otlp_log_tbl` retires.

### 4.2  Add span-events (or an events table)  — **P1**

**What.** Two options; either works.

- **Option A — Nested `events` field on the span row.** Matches the OTel span-events model. Column type `Attrs[]` (JSON array of `{name, timestamp, attributes}`).
- **Option B — Separate `events` table.** Same shape as the logs table minus `severity_*`, keyed by `(trace_id, span_id, event_id)`.

**Why for AI observability.**
- The emitter's `OtlpEvent` rows (business semantics like `order.placed` / `order.failed`) live outside the span-attribute model — they're **discrete moments** in a trace with their own attributes and timestamps. Flattening them into `attrs.*` loses the ordering and the per-event attribute isolation.
- Alerts and drift analyses key off business events (`refund.processed`, `fraud.blocked`) more than off span durations.

**Recommendation:** Option A. Matches OTel's model verbatim, avoids a JOIN for the "spans + their events" query, and doesn't multiply row count.

### 4.3  Token-usage semantic conventions on LLM spans  — producer discipline (no schema change)

**What.** This is a producer-side ask, but it belongs here because it changes how the emitter must be updated. The current `generate_live_telemetry.py` LLM span sets `{"category": "model", "model": …, "gen_ai.prompt": …, "gen_ai.completion": …}` but **does not** set `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` on the span itself — those live on the sibling `Execution` row.

For OliverDB to sum tokens by model / by service via the frozen `input_tokens` / `output_tokens` columns (§1.2), the LLM span must carry them.

**Producer change:**

```python
span("llm", "client", "llm.%s.generate" % model, {
    "category": "model",
    "model": model,
    "gen_ai.system": "google-vertex" if model.startswith("gemini") else "anthropic",
    "gen_ai.request.model": model,
    "gen_ai.prompt": "...",
    "gen_ai.completion": "...",
    "gen_ai.usage.input_tokens": tin,   # NEW
    "gen_ai.usage.output_tokens": tout, # NEW
}, "sp-%s-root" % eid, "ok")
```

Once §1.2 (semantic-convention pull-outs on ingest) lands on OliverDB, these attrs automatically promote to the token columns and become summable — the emitter code doesn't need to know about the column shape.

### 4.4  Trillo-specific `execution_id` field  — producer discipline

**What.** The emitter today writes `executionId` as a top-level field on every OtlpSpan/OtlpLog/OtlpEvent row it produces. OliverDB has no frozen column named that; it belongs in `attrs.trillo.execution_id` (per OTel semantic-convention custom-attribute conventions).

**Not a schema change.** Just: the pod-side helper that formats OTLP → OliverDB puts the `executionId` in `attrs.trillo.execution_id` and it becomes filterable via `attrs.trillo.execution_id = '…'` — indexed like every other attrs key.

### 4.5  Optional: a first-class `execution_id` / `session_id` column  — **P2**

**What.** If real-agent producers uniformly emit a session or execution correlation id, OliverDB could promote it to a frozen column with a sparse+bloom index (like `trace_id`).

**Why.** "All spans for session S" is the second-most-common needle query after `trace_id` lookup. Right now it's `WHERE attrs.session.id = 'S'` — indexed via the attrs bitmap, but a frozen column would be faster and would unlock a session-scoped rollup.

**Cost.** New column = one-time table addition. Producer discipline required for it to be populated.

---

## 5. Multi-tenant scan sharing & sweep amortization

The Trillo AI Agent Observability sweeper fleet runs the SAME queries against different app tenants on a schedule. Today each sweeper hits its own tenant's OliverDB slot; at multi-tenant scale (a Trillo AOS deployment hosting many customer apps) we'd like to run one query and get answers for many tenants from one storage scan, and reduce the total scan-count-per-scheduled-job across the fleet. This section is our formal ask for that class of primitive.

### 5.1  Multi-tenant scan sharing — one scan, N tenants  — **P1**

**What.** A single query issued against N tenant scopes at once, executed as ONE storage scan, returning per-tenant result rows. Two possible shapes for the request:

- **Explicit tenant list.** `POST /v1/multi_query` with body `{tenants: ["acme","initech","umbrella"], sql: "SELECT service_name, count(*) FROM t GROUP BY service_name"}`. Response includes rows tagged by tenant.
- **Implicit via row pin.** A "cluster admin" key whose row pin is a *set* rather than a single value; the engine rewrites the query to `GROUP BY resource_attrs.tenant_id, …` and returns tenant-partitioned rows.

Either shape lands the same win: **N tenants' queries feed off ONE storage scan.**

**Why for Trillo AOS + RLS.**
- Trillo AOS is deploying as a multi-tenant runtime where each customer app is a tenant. Sweepers like `analyze_latency`, `sweep_reliability_health`, `discover_agent_inventory` will run for every tenant on the same cadence. N tenants × the same SQL = N scans today; one scan tomorrow.
- Even in single-tenant Trillo deployments where each tenant has its own OliverDB slot, this primitive helps for the platform's own cross-tenant health dashboards.
- Aligns with columnar analytics' natural strength — one scan feeds many aggregations. `POST /v1/query_batch` already exists for many-queries-one-window; this extends the shape to many-tenants-one-query.

**Suggested behavior.**
- Response includes a per-tenant status: succeeded / policy-denied / no-data. A tenant with a policy denial doesn't sink the whole call.
- Fair queuing: one huge multi-tenant call from customer A shouldn't monopolize the scan for customer B's own workloads.
- Row-pin resolution is server-side; the client doesn't need to enumerate tenant row-pin values.

**Client-side complement (Trillo does today).** Even without this primitive, we can parallelize by sharding tenants across pods — 1-100 in pod 1, 101-200 in pod 2 — and issue one call per shard. That helps but each shard still fires N-per-shard scans against OliverDB. The primitive above collapses that to 1-per-shard.

### 5.2  Sweep-friendly rollups — declarative per-sweep-cadence materialization  — **P2**

**What.** Scheduled jobs on our side (e.g. `sweep_reliability_health` every 5 min, `discover_agent_inventory` every hour) fetch aggregates over rolling windows. Today each run re-scans the raw span table for its window. Instead:

```
POST /v1/rollup {
  "name":       "reliability_5m",
  "cube":       "by_operation",
  "measures":   ["error_count", "duration_us_p95"],
  "tiers_secs": [300, 3600, 86400],
  "warm":       true
}
```

The engine keeps these pre-computed across the tiered windows and refreshes incrementally at ingest, so the sweep query becomes:

```sql
SELECT service_name, error_count, duration_us_p95
  FROM reliability_5m
  WHERE window_start >= now() - '5 minutes'
```

**Why.**
- Scheduled sweeps re-compute the SAME aggregate every N seconds/minutes. Precomputing at ingest amortizes the work over the ingest stream (which is already scanning every row) instead of the sweep loop (which scans on-demand).
- Reduces OliverDB compute cost per scheduled job by a factor of the sweep cadence — a 5-minute sweep over a 24-hour window today re-scans the same 24h × 288 times/day; with precomputed rollups it's 288 O(1) reads.

**Interaction with §5.1.** Multi-tenant + sweep-friendly rollups compose: one precomputed rollup per tenant, one multi-tenant read fetches all N.

### 5.3  Sweep-cadence hints on the client  — **operational, not a product ask**

**Not a request** — this is a client-side pattern we design against, mentioning here so it stays paired with §5.1 / §5.2 in the same review.

- Batch related sweep queries via `/v1/query_batch` (already available) — one storage scan feeds a whole sweeper pass.
- Shard tenants across pods (`1-100 → pod 1`, `101-200 → pod 2`, …) so sweep load parallelizes without any tenant starving.
- Skip sweeps for tenants whose ingest rate over the window is zero — cheaper than running the query and getting an empty result.
- When §5.2 lands: express every sweeper's query as a rollup read; ad-hoc SQL becomes exceptional, not routine.

### 5.4  Denormalization at ingest — pointer to the existing plan

See `oliverdb_refactor_plan.md §7` for the existing enumeration of ingestion-time derivation the OliverDB Rust-plugin model enables — cost, eval pass-rate, canary bucket, per-trace aggregates. That work is complementary to §5.1 and §5.2: precomputing at ingest reduces what each sweep query must compute; multi-tenant scan sharing reduces how many times each sweep query must run. Both compound.

---

## 6. Operational primitives

### 4.1  Batch write ack with per-row errors  — **P2**

**What.** When 950 of 1000 rows in a batch succeed and 50 have invalid types, return `207` (or OTLP partial-success) with per-row error indices. Not "reject all 1000."

**Why.** A single malformed span shouldn't sink a batch of legitimate ones from a healthy exporter.

### 4.2  Observable per-tenant metrics endpoint  — **P2**

**What.** `GET /v1/tenant_metrics` returning ingest rate, buffered rows, flushed rows, query latency p50/p95/p99, error rate — per tenant / per scoped key.

**Why.** Lets Trillo build platform-level dashboards ("is any customer's OliverDB slot near capacity?") without scraping the underlying storage.

### 4.3  Idempotent ingest by client-supplied `batch_id`  — **P2**

**What.** Allow a client to send `batch_id: <uuid>` on `/v1/ingest`. Repeated batches with the same id are deduplicated at ingest.

**Why.** Exporter retries after network failures are the single most common source of dup spans. Exactly-once at the ingest boundary removes an entire dedup pipeline downstream.

---

## 7. Priorities for the near-term roadmap

Ranked by unlock-per-effort for the Trillo AI Agent Observability POC:

| Rank | Item | Reason |
|---|---|---|
| 1 | 1.1 Native OTLP ingest | Every agent framework works without a custom exporter |
| 2 | 4.1 Logs table (OTLP logs signal) | Unblocks the OtlpLog stream — currently stuck in Postgres |
| 3 | 1.2 Semantic-convention pull-outs | Token accounting stops depending on producer discipline |
| 4 | 4.2 Span-events (nested `events` field) | Unblocks the OtlpEvent stream — business-event alerts and drift |
| 5 | 2.1 CTEs | Highest value-per-line-of-code query addition |
| 6 | 3.2 DELETE by predicate | Enterprise-procurement blocker |
| 7 | 1.3 UPSERT for late-arriving spans | Removes exporter-side buffering / crash-loss |
| 8 | 2.2 Broadcast JOINs | Unlocks pricing + service-metadata enrichment |
| 9 | 2.3 Correlated subqueries | Unlocks trace-level and cohort analytics |
| 10 | 1.4 `Text` + `text_match()` | Prompt/response FTS for incident response |
| 11 | 3.1 Per-column retention | Compliance-driven configurability |
| 12 | 5.1 Multi-tenant scan sharing | Trillo AOS + RLS scale: N tenants, 1 scan per sweep |
| 13 | 5.2 Sweep-friendly rollups | Amortizes scheduled-job scan cost across ingest |
| 14 | Everything else | P2 / operational polish |

---

## 8. Open questions to OliverDB

1. **Update / delete semantics.** Is OliverDB append-only today? If so, what's the compaction / tombstone story for §3.2 (DELETE) and §1.3 (UPSERT)?
2. **Plugin surface** (from the mapping-requirements doc). Confirm the per-record vs per-batch API for the Rust plugin; state/cache lifetime; ability to make cheap external lookups (say, an in-process LRU keyed by `agent_name`).
3. **Broadcast-join size threshold.** What's the practical ceiling on a broadcast-joined table (rows × cols) before it needs a different plan?
4. **OTLP compliance test suite.** Willing to run the CNCF OTLP conformance suite once §1.1 lands?
5. **Column addition.** If we ask for a new frozen column (say `session_id`), what's the migration path — table rewrite, dual-write, live?

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-08-22 | Trillo (via Claude Code session) | Initial draft consolidating the 11-item wishlist surfaced during OliverDB scoping. |
| 0.2 | 2026-08-29 | Trillo | Added §5 (multi-tenant scan sharing + sweep-cadence amortization), triggered by the observation during Slice F' dispatcher work that scheduled sweepers all run the same query per-tenant on a fixed cadence. Priorities table gains ranks 12 + 13. |
