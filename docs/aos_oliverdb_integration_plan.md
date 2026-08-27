# AOS ⇄ OliverDB Integration Plan

**Document Version:** 0.2 (draft)
**Status:** Design — ready for slice-AB build. Admin API surface verified live against the tenant on 2026-08-26.
**Companion to:**
- `oliverdb_onboarding.md` — the OliverDB team's tenant onboarding doc (copied verbatim, source of truth for wire shapes)
- `oliverdb_improvements.md` — the OliverDB schema/API wishlist this plan leans on
- `ingest_to_oliverdb_from_agent_frameworks.md` — the parallel design for **customer** agents (ADK / LangChain / CrewAI)
- `Telemetry-Ingestion-Endpoint-Design.md` — the Postgres small-setup ingestion (this doc is the OliverDB counterpart)

Scope: how the **Trillo AOS platform** integrates with OliverDB — for its own emitter (the `generate_live_telemetry` simulator inside `TrilloAgentObservability`), for future AOS-hosted producers, and for Python + Java consumers on the read side.

---

## 1. Architecture in one paragraph

**AOS is on the mint / config path, never on the data path.** A Python function running in an ephemeral Knative execution pod holds OliverDB write credentials for the duration of its invocation (injected as env vars from AOS at pod launch) and writes spans / logs / events **directly** to OliverDB over HTTPS. The pod's outbound-HTTPS allowlist (the "qualified gate") is updated to permit the tenant's OliverDB hostname. Java is only involved in **provisioning** (mint scoped write-only keys, refresh them, pass them into the pod) and in **admin / consumer** APIs (a small Java surface that reads OliverDB on behalf of dashboards, alert rule evaluators, and audits). At the 400 K events/sec throughput target we're planning for, a Java proxy on the write path would be a single-node bottleneck; the pod-direct model scales horizontally by scheduling **more pods** — including per-agent-partitioned pods for parallelism.

---

## 2. Producers

### 2.1  Emitter categories (in scope for AOS integration)

| Producer | Runtime | Purpose | Where it lives |
|---|---|---|---|
| `generate_live_telemetry` | Python function in aos-py-execution pod | Simulator — writes OtlpSpan / OtlpLog / OtlpEvent to Postgres today | `TrilloAgentObservability/.trillo/568/functions/generate_live_telemetry.py` |
| `seed_demo_scenarios` | same | Bootstraps historical demo data | same folder |
| Future: AOS agent runtime | Python (aos-py-execution) | Real agent turns emit spans for tool calls, LLM calls, sub-agents | tcs-ai / aos-py-execution — see the parked `project_agent_telemetry_otel_producer` design |

Customer-hosted producers (ADK / LangChain / CrewAI running in the customer's own env) are **not** in this doc — they're covered in `ingest_to_oliverdb_from_agent_frameworks.md`. They talk to OliverDB directly using their own scoped write-only keys, never routing through Trillo.

### 2.2  Sink parameter

Every emitter function gets a `telemetrySink` parameter that defaults to `postgres`:

```python
def handler(params):
    sink = params.get("telemetrySink", "postgres")   # "postgres" | "oliverdb"
    ...
```

**Routing rule:**

- When `sink == "postgres"`: today's behavior — `ctx.data.create("OtlpSpan"/…)` writes to `otlp_span_tbl` / `otlp_log_tbl` / `otlp_event_tbl` in the app's Postgres schema. `Execution` and fleet rows also go to Postgres. Nothing changes.
- When `sink == "oliverdb"`: **only telemetry rows** (OtlpSpan / OtlpLog / OtlpEvent) route to OliverDB. `Execution` and fleet rows (LogicalAgent, AgentInstance, Application) always go to Postgres — those are entity-shaped, updated, joined, and small-cardinality.

**No dual-write.** Config sets one sink at a time. If the customer wants both (e.g. Postgres for the demo UI + OliverDB for scale testing), they invoke the emitter twice with different `telemetrySink` values.

### 2.3  The `ctx.telemetry` helper

New Python toolkit surface — the function calls one of:

```python
ctx.telemetry.emit_span(**span_fields)
ctx.telemetry.emit_log(**log_fields)
ctx.telemetry.emit_event(**event_fields)
ctx.telemetry.emit_batch(spans=[…], logs=[…], events=[…])
```

The helper picks the sink internally by inspecting the `TRILLO_TELEMETRY_SINK` env var (injected by AOS at pod launch — see §3). When the sink is `postgres`, `emit_span` becomes a `ctx.data.create("OtlpSpan", …)` — same behavior as today. When the sink is `oliverdb`, the helper batches to memory and flushes over HTTPS to OliverDB's `/v1/ingest` endpoint.

**Why a helper (not the function calling `requests.post` directly):**
- One place to add batching, retry, and backpressure.
- One place to enforce the OTel-attribute conventions from `ingest_to_oliverdb_from_agent_frameworks.md` §3.
- Functions stay portable — they don't know which sink they're writing to.

**Why not in the AOS Java tier:** the pod's outbound HTTP is direct to OliverDB (see §1). The helper runs **in the pod's Python process**, holds an in-process batch buffer, and flushes independently. No Java hop on the write path.

### 2.4  Batching + flushing

- **Buffer size:** 1000 spans (or 1 MiB of serialized payload, whichever comes first).
- **Flush cadence:** every 500 ms, or on buffer-full, or on function-return (`atexit`).
- **Retry:** OliverDB's default `202 {"buffered": n}` is fire-and-forget. On a 5xx, exponential backoff up to 60 s, then drop with a metric.
- **`?sync=1` when durable ack is required.** The emitter can opt-in per-batch by calling `ctx.telemetry.flush(sync=True)`. Never on the hot loop.

### 2.5  Partitioning for scale — later phase, design placeholder

At 400 K events/sec, one pod is a bottleneck no matter how fast the helper is. Path forward:

- **Per-agent pod assignment.** Function invocations are already dispatched via Knative; we schedule one pod per logical agent (or per-agent-group), each writing its share of the fleet's telemetry.
- **AOS mints one scoped write-only key per pod** (see §4.3). Row pin: `resource_attrs.service.name = <agent_name>` — the key literally can't write another agent's spans, even if the code tries.
- **OliverDB sees N parallel writers**, each doing 5 K-20 K events/sec — well under the per-connection limit.
- **Fan-in on the emitter side:** for the `generate_live_telemetry` simulator specifically, we split the `PROFILES` dict by agent-name hash across pods; each pod handles its slice.

This is a v2 concern. The v1 slice plan (§8) delivers the single-pod path first.

---

## 3. Credential propagation

### 3.1  Scoped write-only key model

AOS holds the OliverDB **admin key** (in `/Users/anil/workspace2/.claude/.env` for local dev; in Secret Manager for prod). It never travels to a pod.

At function-invocation time:

1. AOS decides whether the function needs OliverDB access (a new bit on `FunctionM.telemetryScopes` or similar — TBD, likely a slice-1 addition to the metadata).
2. If yes, AOS mints (or reuses from a small in-process cache) a **scoped write-only key** for the target agent(s) via `POST /v1/admin/keys` with the admin bearer:

   ```json
   {
     "principal": "aos-app-568-simulator",
     "can_write": true,
     "admin": false,
     "policy": {
       "inject_where": [
         {"dim": "resource_attrs.service.namespace",
          "op": "Eq", "values": ["568"]}
       ]
     }
   }
   ```

   Response: `{"id": "…", "token": "…", …}` — the `token` is what goes into the pod's env; `id` is what we call `DELETE /v1/admin/keys/:id` with on revoke. TTL: 10 minutes (or the pod's max invocation timeout, whichever is smaller). The `inject_where` is enforced by OliverDB's engine — the pod cannot write a span outside its application namespace even if the code tries. **Wire-shape source of truth:** `oliverdb_onboarding.md` §"Scoped keys for your agents" + the admin doc's policy field table.
3. AOS launches the pod with these env vars:

```
TRILLO_TELEMETRY_SINK=oliverdb
OLIVERDB_URL=https://<tenant>.us-west-2.aws.olivercloud.ai
OLIVERDB_SECRET=<the minted scoped key>      # NOT the admin secret
OLIVERDB_KEY_LABEL=<human-readable label, e.g. "svc/generate_live_telemetry/app-568/2026-08-22T12:00Z">
```

4. The pod's Python process reads them at import time (in the toolkit helper), initializes the batch buffer, and it's ready.

Once the invocation completes and the pod recycles, the scoped key expires. Even if it were somehow exfiltrated, its write scope is bounded to that application, and it can only INSERT, never read or admin.

### 3.2  Key-cache in AOS

Minting a scoped key on every invocation would add ~50-100 ms of OliverDB round-trip to pod launch. Instead:

- AOS keeps an in-process cache of scoped keys, keyed by `(applicationId, purpose)` — say `("568", "simulator")`.
- Cache entry TTL: 8 minutes (2-minute jitter below the key's own 10-minute TTL, so we always refresh with room to spare).
- Cache hit → reuse the existing key in the pod launch env. Cache miss → mint fresh from the admin key, populate cache.
- On process restart AOS re-mints. That's fine — it's a short blip, and the old keys expire on their own.

### 3.3  Rotation

Admin key rotation is on the OliverDB console. When rotated, AOS restarts (or re-reads `.env` / secret) — trivial. Scoped keys have a 10-minute TTL and roll over continuously, so no explicit rotation logic on our side.

---

## 4. Pod egress — updating the HTTPS qualified-gate

Per the Aug 21 decisions on pod de-privilege (memory: `project_aos44_pod_deprivilege`), the aos-py-execution pod locks outbound egress via a **qualified HTTPS target list**. OliverDB isn't on it today; the toolkit's HTTPS helper would refuse the write.

### 4.1  What must change

- **Add the customer's OliverDB hostname** to the pod's outbound allowlist. Format: fully-qualified DNS name, TLS-verified.
- Because the hostname is **customer-specific** (each tenant has their own OliverDB slot at `https://<tenant>.<region>.olivercloud.ai`), the allowlist can't be a static string — it must be per-application.

### 4.2  How

Two options; I recommend the second.

- **Option A — static allowlist entry per env.** Hardcode `*.olivercloud.ai` (subdomain wildcard). Simple, but too coarse — a compromised pod could hit *any* OliverDB tenant, not only the tenant it's scoped to.
- **Option B (recommended) — dynamic allowlist from AppConfig.** AOS reads `AppConfig.analyticsDbUrl` for the invoking app and injects it as an env-var-driven allowlist entry at pod launch:

```
POD_HTTPS_ALLOWLIST=aos.trillo.io,api.gemini.googleapis.com,<tenant>.us-west-2.aws.olivercloud.ai
```

The Python toolkit's HTTPS helper reads this env var on startup and permits only those hosts. Per-app allowlist, per-pod. Compromised pod = compromised only for that tenant.

### 4.3  New AppConfig fields — vendor-neutral by design

Add `AppConfig.analyticsDbUrl` (String, nullable). Populated at app-deploy time (or via a small admin UI). When null, resolve via the env-level fallback `tcs.analytics.db.default.url`; when both null, the pod runs in Postgres-sink mode regardless of `telemetrySink` requests.

Related: `AppConfig.analyticsDbEnabled` (Boolean, default false) — explicit off-switch that overrides `telemetrySink=oliverdb` requests. Belt-and-braces for the case where a customer wants Postgres-only writes even if the URL is set.

**Why `analyticsDb*` and not `oliverdb*`.** The URL field is *what the app points at for its columnar/analytical big-data store*. Today the only thing on the other end is OliverDB, but the field's semantics — "a big-data store the app writes analytics to" — apply equally to ClickHouse, Tempo, Grafana Cloud, or a future OliverDB successor. Naming it vendor-neutral means we don't rename the ClassM field the day we integrate a second backend; we add an `AppConfig.analyticsDbKind` discriminator (`oliverdb`|`clickhouse`|…) and route to the right client. **Vendor-specific bits stay vendor-specific**: the admin credential env var is `OLIVERDB_ADMIN_SECRET` (not `ANALYTICS_DB_ADMIN_SECRET`) because auth is a per-vendor concept; same for the pod's `TRILLO_TELEMETRY_SINK=oliverdb` — the *write path* is vendor-specific. Only the *config surface* is neutral.

Resolution order at pod-launch time:

1. `AppConfig.analyticsDbUrl` (per-app override — this is the sharding lever)
2. `tcs.analytics.db.default.url` (env-level default, e.g. `https://trillo.us-west-2.aws.olivercloud.ai` in dev)
3. If both null → analytics-sink disabled for this pod.

> **Note on AppConfig persistence** (memory `project_appconfig_deploy_persist_gotcha`): every new `AppConfig.*` field must be added to `AOS DeployAppMetadata.bootstrapAppConfig` **and** to the hand-written INSERT branch, or deploy silently drops the value. Booleans need the `multiTenant` pattern, not `copyIfPresent`. Slice §8.2 accounts for this.

---

## 5. Read side

### 5.1  Python read helper

A parallel toolkit surface:

```python
rows = ctx.telemetry.query("SELECT service_name, count(*) FROM t "
                          "WHERE ts > $1 GROUP BY service_name", [ago("1h")])
```

Under the hood it's a `POST /v1/query` (or `/v1/query_batch` for dashboards). The helper:
- Reads a **read-scoped key** (§5.3) from env, distinct from the write key.
- Applies statement timeouts (matches the AOS Postgres query-hardening we already do).
- Formats results as a list-of-dicts, matching `ctx.data.query`'s shape so downstream code is portable across sinks.

Consumers today: `sweep_reliability_health`, `sweep_governance_audit`, `detect_behavioral_drift`, `analyze_latency`, `get_top_token_consumers`, `alert_rule_evaluator`, `run_sweeper_pass` (from the TrilloAgentObservability app). Each currently does `ctx.data.query("OtlpSpan", …)` against Postgres. Migration path in §5.4.

### 5.2  Java read helper

Some AOS admin/analytics jobs (e.g. platform-level `/admin/observability/*` endpoints, alert scheduler evaluations) run in Java. A small `OliverDbClient` service in tcs-service:

```java
public interface OliverDbClient {
  Result query(long appId, String sql);         // read-side scoped key
  Result queryBatch(long appId, List<String> sqls);
  Result ingest(long appId, List<Span> spans);  // admin/tenant-provisioning use only
}
```

The `OliverDbClient` reads the admin key from platform config, mints scoped keys as needed, and caches per-app similar to §3.2. Not on the hot data path — only on Java-side admin flows.

### 5.3  Read-scoped keys

Same primitive as write-scoped keys, different scope:
- `SELECT`-only privilege
- Row pin: `resource_attrs.service.namespace = <application_id>`
- Optional column redaction (the `body` Text column scrubbed for non-PII-authorized consumers — depends on `oliverdb_improvements.md §3.4`)
- TTL 15 min, cached similarly to the write path

The SRE Copilot agent gets its own scoped read key with tighter scope (see `SRE-Copilot-Tool-Manifest.md`) — that's out of scope for this AOS plan.

### 5.4  Postgres → OliverDB read migration

Consumer functions (§5.1 list) currently `ctx.data.query("OtlpSpan", filters=[…])`. With OliverDB in the picture the migration options are:

- **Option A — SQL over OliverDB.** Rewrite each consumer to use `ctx.telemetry.query("SELECT …")`. Cleanest long-term; each function gets faster and gains group-by / percentile support.
- **Option B — Dual read via a dispatcher.** Keep `ctx.data.query("OtlpSpan", …)` semantics; the toolkit routes to OliverDB or Postgres based on `TRILLO_TELEMETRY_SINK`. Zero function changes; loses OliverDB's SQL power (`ctx.data.query` is a limited surface — no group-by beyond `groupBy: […]`, no percentile, no time_bucket).

**Recommendation: Option A** — the whole point of moving to OliverDB is the SQL surface. Wrap the migration into slice §8.4.

---

## 6. Observability of the integration itself

The integration must observe itself, or its failures become invisible.

- **Metrics.** Per-pod: spans/s emitted, spans/s dropped, batch size histogram, flush latency p50/p95/p99, OliverDB 5xx count. Emitted via… OliverDB itself (as spans with `service.name=trillo.telemetry.helper`). Meta.
- **Admin dashboard.** A `/admin/observability/health` endpoint on tcs-ai returns aggregate write health across pods (last-N-minutes drop rate, active scoped-key count, OliverDB error rate).
- **Alerts.** Pager: drop rate > 1% for 5m, OliverDB 5xx rate > 10% for 2m, admin-key close to expiration.

---

## 7. Java-side vs Python-side responsibility map

| Concern | Language | Where |
|---|---|---|
| Admin key custody | Java (AOS platform) | tcs-service, read from Secret Manager / `.env` |
| Scoped-key minting (write) | Java | new `OliverDbAdminService` in tcs-service |
| Scoped-key minting (read) | Java | same |
| Scoped-key cache | Java | in-process LRU |
| Injecting keys into pods | Java | existing pod-launch path (extend env-var set) |
| Pod HTTPS allowlist config | Java (writes AppConfig) → Python (reads env var) | AppConfig field + pod init |
| Batching + flushing to OliverDB | Python | new `ctx.telemetry.*` in aos_toolkit |
| Consumer read queries | Python (mostly) + Java (admin) | aos_toolkit + `OliverDbClient` |
| Metrics / self-observability | Python (emit) + Java (aggregate) | toolkit + tcs-service admin endpoint |
| Dashboard rendering | Java + Angular UI | tcs-ai / Trillo AI UI |

---

## 8. Slice plan

Numbered slices, each self-contained. Order = dependency order.

### 8.1  Slice AB — config plumbing + scoped-key minting *(combined; prereq for everything)*

Combined because the OliverDB admin API is now known to be a live REST surface (`POST /v1/admin/keys` etc., verified 2026-08-26). Splitting A from B would have shipped stub code in A that B immediately replaces; one slice avoids the throwaway.

**Config plumbing:**
- Add `AppConfig.analyticsDbUrl` (String, nullable) + `AppConfig.analyticsDbEnabled` (Boolean, default false) fields (per §4.3, including `DeployAppMetadata.bootstrapAppConfig` gotcha for both fields; boolean uses the `multiTenant` pattern).
- Add platform config: `tcs.analytics.db.default.url` (env-level fallback URL) and `tcs.oliverdb.admin.secret` (from `OLIVERDB_ADMIN_SECRET` env). Wired through `PlatformConfig` / `application-{env}.yml`.
- Admin UI: two new fields in the AppConfig editor.

**Scoped-key minting service:**
- `OliverDbAdminService` in tcs-service. Interface:
  ```java
  Result mintWriteKey(long appId, String purpose);   // returns {token, id, ttlSeconds}
  Result mintReadKey (long appId, String purpose);
  Result revokeKey   (String keyId);
  Result health      (long appId);                    // pings /v1/admin/keys via app's URL
  ```
- Impl calls the concrete REST surface documented in `oliverdb_onboarding.md` §"Admin API" — `POST /v1/admin/keys` with the policy shape in §3.1 of this doc.
- Resolves the URL per §4.3 order (AppConfig → env default).
- In-process LRU cache keyed by `(appId, purpose, kind)`, 8-minute TTL.
- `/admin/oliverdb/health` GET endpoint — invokes `.health(appId)` for admin smoke tests.
- Unit tests + one integration test against the live dev tenant (guarded behind an env-var).

- **Deliverable:** Java can mint + revoke scoped keys against the live OliverDB tenant; smoke-tested via `/admin/oliverdb/health`.

### 8.2  Slice C — Pod launch: inject OliverDB env + allowlist
- Extend the pod-launch env-var set to include `TRILLO_TELEMETRY_SINK`, `OLIVERDB_URL`, `OLIVERDB_SECRET`, `POD_HTTPS_ALLOWLIST`.
- Compute allowlist from AppConfig (§4.2 Option B).
- On the Python side, the toolkit's existing HTTPS gate reads `POD_HTTPS_ALLOWLIST` and permits those hosts. **Update the qualified-gate implementation** if it's currently a hardcoded list — refactor to accept the env var. This is the "recently implemented qualified gates" hook the user flagged.
- **Deliverable:** a pod launched for an Oliver-enabled app can reach the OliverDB hostname (verified by a smoke curl inside the pod).

### 8.3  Slice D — `ctx.telemetry` helper (Python)
- New module `aos_toolkit/telemetry.py` — `emit_span`, `emit_log`, `emit_event`, `emit_batch`, `flush`, `query`, `query_batch`.
- Postgres sink path: delegates to existing `ctx.data.create` / `ctx.data.query`.
- OliverDB sink path: in-process buffer (1000 spans / 1 MiB / 500 ms), HTTP POST to `/v1/ingest` with the semantic-convention conforming shape from `ingest_to_oliverdb_from_agent_frameworks.md §3`.
- **Deliverable:** any function can `ctx.telemetry.emit_span(...)` and it lands in the right place.

### 8.4  Slice E — Update `generate_live_telemetry` emitter
- Add `telemetrySink` param (default `postgres`).
- Replace direct `ctx.data.create("OtlpSpan"/"OtlpLog"/"OtlpEvent", …)` with `ctx.telemetry.emit_*`.
- Add `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` and `gen_ai.system` / `gen_ai.request.model` on the `llm` span (per `oliverdb_improvements.md §4.3`).
- Add `trillo.execution_id` in `attrs`, remove the top-level `executionId` field (per `§4.4`).
- Update `seed_demo_scenarios` and `process_execution_telemetry` similarly.
- **Deliverable:** the same simulator can populate Postgres or OliverDB from one invocation.

### 8.5  Slice F — Java read client
- `OliverDbClient` service in tcs-service, wired from the mint service in Slice B.
- Small `/admin/observability/query` endpoint (JSON in, JSON out) for internal admin tooling.
- **Deliverable:** Java-side callers can query OliverDB with a scoped read key.

### 8.6  Slice G — Migrate first sweeper to OliverDB
- Pick `sweep_reliability_health` (or another single sweeper) as the pilot.
- Rewrite from `ctx.data.query("OtlpSpan", …)` to `ctx.telemetry.query("SELECT … FROM t WHERE …")`.
- Verify output matches Postgres-sink runs.
- **Deliverable:** one production sweeper runs against OliverDB end-to-end.

### 8.7  Slice H — Migrate remaining sweepers + analyzers
- `sweep_governance_audit`, `detect_behavioral_drift`, `analyze_latency`, `get_top_token_consumers`, `alert_rule_evaluator`, `run_sweeper_pass`.
- Each one PR-sized; sequence by risk.
- **Deliverable:** all read consumers run against OliverDB.

### 8.8  Slice I — Self-observability
- Wire the toolkit helper to emit its own spans (`service.name=trillo.telemetry.helper`).
- Admin dashboard tile in Trillo AI UI.
- Basic alert rules.
- **Deliverable:** OliverDB write health is visible + alerted.

### 8.9  Slice J (v2) — Partitioning for scale
- Deferred. Kicks in when we approach the single-pod ceiling.
- Design: per-agent pod pool with per-agent scoped keys and `service.name` row pins.

---

## 9. Open questions

1. **OliverDB scoped-key mint API.** Do they expose a REST endpoint the Java admin service calls (`POST /admin/keys`), or only their console UI? If UI-only in the near-term, we may need a manual pre-provisioning step + AOS reads keys from Secret Manager for slice A/B.
2. **`AppConfig.analyticsDbUrl` per-env.** Multi-env AOS (memory: `project_multi_env_model`) has per-env config. Does OliverDB URL differ by env, or is one URL shared across envs? If per-env, the field lives on `AppEnv` not `AppConfig` — small refactor.
3. **Simulator concurrency.** Does the `generate_live_telemetry` simulator today run as one function call producing N spans sequentially, or is there already a batch/parallel model? If sequential, the OliverDB batch buffer masks the slowness; if the user wants real per-agent parallelism in v1 (not v2), slice §8.10 moves up.
4. **`ctx.data.query` compatibility.** For consumers we don't want to migrate immediately, is a dispatcher (Option B in §5.4) worth building as a fallback, or do we commit to the pure-SQL migration path?
5. **Java consumer scope.** Beyond admin / alert-scheduler use, is there a plan for Java-side app code (not admin) to read observability data? If yes, we may need a general-purpose Java toolkit surface, not only an admin client.
6. **Metrics-signal support.** OTLP has three signals (traces, logs, metrics). This plan covers traces + logs + events. Are OTLP metrics in scope for this integration, or a v2 concern? OliverDB's current single-table shape doesn't support metrics well, so answering this feeds back into `oliverdb_improvements.md`.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-08-22 | Trillo (via Claude Code session) | Initial draft. Architecture pivoted from Java-proxy to pod-direct-write per user constraint (400 K events/sec target). Ten-slice plan; slice J deferred. |
