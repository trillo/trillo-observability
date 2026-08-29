# Using OliverDB with Trillo AOS — Application Team Guide

**Document Version:** 0.1
**Audience:** Application developers writing Trillo AOS apps.
**Companion to:**
- `aos_oliverdb_integration_plan.md` — the platform-side integration.
- `oliverdb_refactor_plan.md` — the reference examples in `TrilloAgentObservability`.
- `oliverdb_onboarding.md` — the OliverDB team's tenant onboarding.

This is a practical guide. Copy-paste-modify the examples. When something is unclear, look at the referenced function in `TrilloAgentObservability/.trillo/568/functions/` — those files are the working, shipped examples of every pattern here.

---

## 1. When to use OliverDB, when to stay on Postgres

Trillo AOS uses two stores. Pick by the shape of the data, not by preference.

| Data type | Store | Why |
|---|---|---|
| **Telemetry** — spans, logs, events | **OliverDB** | High-volume, append-only, analytical queries with `GROUP BY`, `time_bucket()`, percentiles |
| **Business entities** — users, orders, agents, applications, metric rollups, findings | **Postgres** | Row-level updates, joins across tables, transactional writes |
| **Metric rollups & aggregations** you write to | **Postgres** | Same as entities |

Rule of thumb: **if you'd write `UPDATE`, it goes to Postgres.** OliverDB is append-only columnar and doesn't support row-level updates.

You don't have to choose one for a whole function. The reference implementations write derived `Execution` and `MetricRollup` rows to Postgres in the same function that queries spans from OliverDB. That's the intended shape.

---

## 2. Turning on OliverDB for your app

Two `AppConfig` fields drive everything. Both are edited from Trillo AOS Designer (the AppConfig editor screen).

| Field | Type | Meaning |
|---|---|---|
| `analyticsDbUrl` | string | Your OliverDB tenant URL, e.g. `https://acme.us-west-2.aws.olivercloud.ai`. Blank means "use the platform default from `tcs.analytics.db.default.url`." Both blank means analytics-sink is disabled for this app. |
| `analyticsDbEnabled` | boolean | Master switch. Even with a URL set, `false` here means AOS doesn't inject any OliverDB credentials into function invocations, and any function that requests `sink="oliverdb"` will raise. |

**Contract:**
- Both must be true+set for OliverDB to work for your app.
- The URL is a per-app override. Leave it blank in dev to use the platform's shared dev tenant.
- Flip `analyticsDbEnabled` to `false` any time to disable OliverDB for your app WITHOUT clearing the URL. Handy for experimentation.

**How AOS uses these:**
- At function invocation, if `analyticsDbEnabled=true` AND a URL resolves, AOS mints two scoped OliverDB keys (one write-only, one read-only) row-pinned to your app's `service.namespace`. The keys travel to the pod as request headers on the `/event` or `/execute` invocation. The pod exposes them via `aos_toolkit.get_analytics_db_config()`; you don't touch that directly.

---

## 3. Writing a function that emits telemetry

Every function has a single API surface: **`ctx.telemetry`**. Three emit methods:

```python
ctx.telemetry.emit_span(...)    # OTel spans -- can go to OliverDB
ctx.telemetry.emit_log(...)     # OTel logs  -- Postgres today (OliverDB pending)
ctx.telemetry.emit_event(...)   # OTel events -- Postgres today (OliverDB pending)
```

### 3.1 The sink kwarg — one line, three outcomes

Every emit method takes an optional `sink` kwarg:

```python
ctx.telemetry.emit_span(..., sink=None)         # or omit -> Postgres (default)
ctx.telemetry.emit_span(..., sink="postgres")   # explicit -> Postgres
ctx.telemetry.emit_span(..., sink="oliverdb")   # OliverDB, or raises AOSException
```

**Behavior when `sink="oliverdb"` and OliverDB isn't reachable:**
- App has `analyticsDbEnabled=false` OR no URL resolves → raises `AOSException` immediately, with a message explaining which fix to apply.
- **No silent fallback.** If you want Postgres, ask for it explicitly with `sink="postgres"`.

**For `emit_log` and `emit_event` — sink="oliverdb" raises TODAY** because OliverDB's tenant schema doesn't yet have logs/events tables (tracked in `oliverdb_improvements §4.1/§4.2`). Once those ship, `sink="oliverdb"` will start working; your code doesn't need to change.

### 3.2 The developer pattern — one line at the top

Copy-paste this pattern into any function that emits telemetry:

```python
def handler(params):
    sink = params.get("telemetrySink", "postgres")

    ctx.telemetry.emit_span(
        trace_id=trace_id,
        span_id=span_id,
        span_name="agent.run",
        kind="SERVER",
        status="OK",
        service_name="my-agent",
        execution_id=execution_id,
        attributes={"gen_ai.request.model": "claude-sonnet-5"},
        resource_attributes={
            "service.namespace": str(ctx.app_id),   # scoped-key row-pin key
            "trillo.application.name": "my-app",
        },
        sink=sink,   # <-- the one lever
    )
```

Copy the function, change the default `"postgres"` to `"oliverdb"`, or pass `telemetrySink="oliverdb"` in the params at invoke time. Either way, no env or admin change is required.

### 3.3 Producer discipline you should follow

If your spans will ever live in OliverDB, follow these rules — they cost nothing but pay off later:

| Attribute | Where | Why |
|---|---|---|
| `resource_attributes["service.namespace"] = str(ctx.app_id)` | Every span | Matches the scoped read key's row-pin. Without this, your app's scoped read keys will filter out your own rows. |
| `attrs["gen_ai.system"]` | LLM spans | Pulled into OliverDB's frozen columns (once §1.2 lands). |
| `attrs["gen_ai.request.model"]` | LLM spans | Same. |
| `attrs["gen_ai.usage.input_tokens"]` / `["gen_ai.usage.output_tokens"]` | LLM spans | Same — token accounting depends on these. On OliverDB, the `emit_span` `input_tokens=` / `output_tokens=` kwargs put them in first-class columns automatically. |
| Trillo execution correlation | `execution_id=` kwarg | Both sinks get `attrs.trillo.execution_id`; Postgres additionally gets the top-level `executionId` column for backwards compat. |

**Reference implementation:** `TrilloAgentObservability/.trillo/568/functions/generate_live_telemetry.py` — the largest emitter, with every rule above applied. Copy this file and modify.

### 3.4 What about `emit_log` and `emit_event` in an OliverDB-target function?

For now, emit logs and events **without** a sink kwarg (they land in Postgres), even when your spans go to OliverDB. Comment locally:

```python
# span goes to whichever sink params picks:
ctx.telemetry.emit_span(..., sink=sink)

# logs/events always Postgres until OliverDB adds those tables:
ctx.telemetry.emit_log(...)
ctx.telemetry.emit_event(...)
```

When OliverDB ships §4.1 (logs) and §4.2 (events) upstream, add `sink=sink` to those calls too. Nothing else needs to change.

---

## 4. Writing a function that reads telemetry

Reading is where the shape difference between the two stores matters — Postgres uses `ctx.data.query("OtlpSpan", filters=[...])` (dict filters), OliverDB uses `ctx.telemetry.query(sql)` (full SQL). We use the **dispatcher pattern** so callers can choose.

### 4.1 The dispatcher pattern

Every reader function that supports both sinks follows this shape:

```python
def handler(params):
    params = params or {}
    if params.get("telemetrySink") == "oliverdb":
        return _handler_oliverdb(params)
    return _handler_postgres(params)


def _handler_postgres(params):
    """Existing code, unchanged. Uses ctx.data.query."""
    ...


def _handler_oliverdb(params):
    """SQL-first implementation. Uses ctx.telemetry.query."""
    ...
```

Both handlers **must return the same shape** — same fields, same types. Add a `"sink"` field to the returned data so an A/B invocation can distinguish which ran:

```python
return ctx.success(data={
    "executionsProcessed": ...,
    "dependenciesCreated": ...,
    "sink": "postgres",   # or "oliverdb"
})
```

Callers never branch on sink; the shape parity means every downstream consumer treats the two results identically.

### 4.2 The `ctx.telemetry.query` surface

```python
rows = ctx.telemetry.query(
    "SELECT service_name, count(*) AS n FROM t "
    "WHERE ts > 1700000000000 AND ts < 1700003600000 "
    "GROUP BY service_name "
    "LIMIT 100"
)
# rows is list[dict[str, Any]]
```

**Important OliverDB SQL notes:**
- Single-table query — the spans table is named `t`.
- Attrs-key filter/select — `attrs.gen_ai.request.model`, `resource_attrs.service.namespace`, etc.
- `ts` is the implicit column (epoch milliseconds).
- `time_bucket('5m', ts)`, `percentile_cont(0.95) WITHIN GROUP (ORDER BY duration_us)` — supported.
- **No JOINs, no CTEs, no correlated subqueries** — merge with Postgres data in Python.
- `LIMIT` is your friend.

Full syntax reference: `oliverdb_onboarding.md`.

**Errors are loud:**
- No OliverDB config bound → `AOSException("no OliverDB config is bound to this invocation…")`
- Read-key mint failed platform-side → `AOSException("OliverDB write config is bound but no read_token was provisioned…")`
- Upstream 4xx/5xx from OliverDB → `AOSServerError` with the response body attached.

None of these are silently caught; wrap in `try/except` if your function should degrade gracefully.

### 4.3 Reference implementations

Five reader functions in `TrilloAgentObservability/.trillo/568/functions/` follow the dispatcher pattern. Pick the one closest to what you're building:

| Function | Pattern | Good example of |
|---|---|---|
| `calculate_execution_cost_and_tokens.py` | Small trigger handler, one execution at a time | Simplest dispatcher; per-invocation SQL |
| `reconcile_applications.py` | Sweeper with per-agent lookup | Callable-injection factoring; `LIMIT 1` resource-attrs pattern |
| `validate_dataset.py` | Validation checks with just one OTLP probe | When most of the function is Postgres-native and only one check touches OTLP |
| `reconcile_agent_dependencies.py` | Windowed sweep | ONE SQL round-trip via `ts` window filter — replaces N per-execution queries |
| `discover_agent_inventory.py` | Big analytical sweep | Everything the handler needs in one SQL query, downstream code shape-invariant |

---

## 5. Common failure modes and how to fix them

### 5.1 `AOSException: OliverDB sink requested but no analytics-DB config bound…`

You called `ctx.telemetry.emit_span(sink="oliverdb")` (or `sink="oliverdb"` on log/event) and either:
- The app has `AppConfig.analyticsDbEnabled=false`.
- No URL resolves — `AppConfig.analyticsDbUrl` is blank AND the platform's `tcs.analytics.db.default.url` is not configured.
- The URL is set but the OliverDB slot itself rejected the mint request (rare).

**Fix:** flip `analyticsDbEnabled=true` in Designer's AppConfig editor and set a URL if you don't have one. Or change the emit call to `sink="postgres"` if you want to fall back explicitly.

### 5.2 `AOSException: ctx.telemetry.query requires an OliverDB read-key context…`

Your `_handler_oliverdb` called `ctx.telemetry.query(sql)` but the invocation doesn't have OliverDB context bound. Same root cause as 5.1.

**Fix:** same as 5.1 — enable the app, or don't call the OliverDB handler when the app isn't configured. In practice, if a caller passes `telemetrySink="oliverdb"` for an app that isn't configured, this is the actionable error they'll see. Point them at Designer.

### 5.3 `AOSException: OliverDB write config is bound but no read_token was provisioned…`

Writes work but reads don't. The AOS platform side's read-mint failed (network glitch, OliverDB slot rejecting the mint policy briefly).

**Fix:** retry the invocation. Read-keys are per-invocation and get freshly minted. If it persists, check platform logs for `OliverDbAdminService.mintReadKey` warnings.

### 5.4 Empty results from `ctx.telemetry.query`

You get `[]` back on a query that should return data. Two things to check:

- **Row-pin mismatch.** Your producer set `resource_attrs.service.namespace` to something other than `str(ctx.app_id)`. The scoped read key filters out rows that don't match. Fix the producer.
- **Time window off.** OliverDB's `ts` is in epoch milliseconds; if you passed seconds or nanos, the filter will match nothing. Double-check your unit.

### 5.5 Postgres path works, OliverDB path returns different numbers

**Shape parity is a hard requirement** — the two handlers should return byte-equivalent output. If they differ:

- Different time windows? (One filter is inclusive, the other exclusive?)
- Different tokens counting rule? (E.g., Postgres path reads `attrs.gen_ai.usage.*` and the OliverDB path reads the first-class `input_tokens` column — check whether the producer populates both.)
- Null handling. Postgres path might treat `null` differently than OliverDB.

Run both invocations back-to-back on a fixed dataset, `diff` the outputs, find the first mismatch. Fix the SQL until parity holds.

---

## 6. UI considerations

If your app has a UI (Trillo AOS Designer or your own frontend) that invokes these functions and displays results:

### 6.1 Passing `telemetrySink` from UI

The `telemetrySink` param is a plain string in the invoke body. Add a small selector — "OliverDB (fast, columnar)" vs "Postgres (all history)" — for pages that support both. Or default to `oliverdb` in prod and `postgres` in dev; your call.

### 6.2 Shape parity means one renderer

If you followed §4.1's shape-parity contract, the same UI component renders both sink outputs. Don't put sink-specific logic in the UI. If a numbers differ, that's a bug in the function; fix the function.

### 6.3 Latency expectations

- OliverDB path: single SQL, usually sub-second even over 24-hour windows.
- Postgres path: for small windows, similar. For large windows or per-execution lookups, N Postgres round-trips add up — expect seconds.

If your UI has a "load more" or paginated result set, prefer the OliverDB path if configured — its `LIMIT` behavior is cheap. On Postgres, pagination requires care not to trigger a full-window scan per page.

### 6.4 Dashboards that read OliverDB directly

For a UI that queries OliverDB directly (not via a function), you'd need a scoped read key delivered to the browser. **Don't do this.** The current model is: functions run on the pod with per-invocation-minted keys; the browser talks to your functions, not to OliverDB. Browser-side OliverDB access is a separate design that hasn't been built.

---

## 7. Quick reference — every pattern in one place

### 7.1 Emit a span

```python
ctx.telemetry.emit_span(
    trace_id=trace_id,
    span_id=span_id,
    span_name="agent.run",
    kind="SERVER",
    status="OK",
    service_name="my-agent",
    start_time_unix_nano=start_ns,
    end_time_unix_nano=end_ns,
    input_tokens=42,
    output_tokens=17,
    execution_id=eid,
    attributes={
        "gen_ai.system": "anthropic",
        "gen_ai.request.model": "claude-sonnet-5",
    },
    resource_attributes={
        "service.namespace": str(ctx.app_id),
        "trillo.application.name": "my-app",
        "environment": "production",
    },
    sink=params.get("telemetrySink", "postgres"),
)
```

### 7.2 Emit a log or event

```python
ctx.telemetry.emit_log(
    body="handler completed",
    severity="INFO",
    trace_id=trace_id,
    span_id=span_id,
    execution_id=eid,
    log_id=f"log-{eid}",
    timestamp_ms=int(time.time() * 1000),
    attributes={"outcome": "success"},
    # no sink kwarg -- always Postgres today
)

ctx.telemetry.emit_event(
    name="order.placed",
    trace_id=trace_id,
    span_id=span_id,
    execution_id=eid,
    event_id=f"ev-{eid}",
    attributes={"status": "ok"},
    # no sink kwarg
)
```

### 7.3 Query OliverDB from a function

```python
rows = ctx.telemetry.query(
    "SELECT service_name, count(*) AS n, avg(duration_us) AS avg_dur "
    "FROM t "
    f"WHERE ts >= {int(start_ms)} AND ts <= {int(end_ms)} "
    "GROUP BY service_name "
    "ORDER BY n DESC "
    "LIMIT 20"
)
for r in rows:
    print(r["service_name"], r["n"], r["avg_dur"])
```

### 7.4 Dispatcher skeleton

```python
def _handler_postgres(params):
    # existing code, unchanged
    return ctx.success(data={..., "sink": "postgres"})

def _handler_oliverdb(params):
    rows = ctx.telemetry.query("SELECT ... FROM t WHERE ...")
    # process rows, return same shape
    return ctx.success(data={..., "sink": "oliverdb"})

def handler(params):
    params = params or {}
    if params.get("telemetrySink") == "oliverdb":
        return _handler_oliverdb(params)
    return _handler_postgres(params)
```

---

## 8. What's not in scope for now

Things that come up in questions but aren't ready yet:

- **OliverDB logs / events tables.** Pending `oliverdb_improvements §4.1/§4.2`. Emit calls with `sink="oliverdb"` on logs/events raise until then.
- **OliverDB metrics table.** Not confirmed by OliverDB team yet. No `emit_metric` in the toolkit.
- **Cross-tenant SQL from a single function.** Even a "platform admin" function that wants to query many apps' data at once — the scoped read key is per-invocation, per-app. Cross-tenant analytics is future work, tracked in `oliverdb_improvements §5.1`.
- **Browser → OliverDB directly.** All OliverDB access flows through pod functions. UIs invoke your function, your function reads OliverDB.
- **JOINs, CTEs, subqueries in `ctx.telemetry.query`.** OliverDB doesn't support them yet (`oliverdb_improvements §2`). Merge with Postgres data in Python for now.

---

## 9. Contacts

- Backend / platform integration questions: this integration's Trillo platform team.
- OliverDB SQL / wire questions: `oliverdb_onboarding.md` is the authoritative reference; hard questions go to the OliverDB team.
- Reference implementations: `TrilloAgentObservability/.trillo/568/functions/` — every pattern in this guide has a working example there.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-08-29 | Trillo (via Claude Code session) | Initial application-team guide. Covers ctx.telemetry emit + query surfaces, the sink kwarg, the reader dispatcher pattern, and UI integration. Points at the five Slice F' + two Slice E reference implementations. |
