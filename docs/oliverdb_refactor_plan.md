# TrilloAgentObservability — OliverDB Refactor Plan

**Document Version:** 0.1 (draft)
**Status:** For internal review before slice work begins.
**Companion to:**
- `aos_oliverdb_integration_plan.md` — the platform-side integration (Slices AB / C / D / E already shipped as of 2026-08-27).
- `oliverdb_improvements.md` — the OliverDB wishlist; §4.1 (logs table) and §4.2 (events table) are what gate log/event migration.
- `oliverdb_onboarding.md` — the OliverDB team's tenant onboarding doc.

Scope: how each of the ~56 functions in `TrilloAgentObservability/.trillo/568/functions/` should treat the two data stores (Postgres + OliverDB), and the slice order to get there.

---

## 1. Ground rules (from the 2026-08-27 design conversation)

**Postgres remains the destination for aggregate writes.** `MetricRollup`, `PlatformFinding`, `AnalysisBaseline`, `Dependency`, `Application`, `AgentInstance`, `LogicalAgent`, `Alert`, `GovernanceEvaluation`, `GovernancePolicy`, `SweeperRun`, and all other entity tables continue to live in Postgres — updates, joins, transactional writes.

**OliverDB is the source of truth for telemetry.** Spans (and eventually logs + events + optional metrics) that today land in `OtlpSpan` / `OtlpLog` / `OtlpEvent` route to OliverDB when the app opts in. Postgres remains the default sink and the fallback source for consumers that haven't been rewritten.

**No forced migration.** The default sink is `postgres` everywhere. Developers opt into OliverDB per-invocation via the `telemetrySink` parameter — no admin toggles, no env changes required for a developer to A/B compare a rewrite.

**Errors are loud, not silent.** `sink="oliverdb"` on any emit call, and `telemetrySink="oliverdb"` on any dispatcher, raises `AOSException` with actionable text when the OliverDB config is not bound. This matches the Slice E writer semantics and avoids the "why is my OliverDB dashboard empty?" mystery.

---

## 2. The two patterns

### 2.1 Writers — Slice E's sink kwarg (no dispatcher)

For functions that emit OTLP data. Same code path, sink is a kwarg on each emit.

```python
def handler(params):
    sink = params.get("telemetrySink", "postgres")
    ctx.telemetry.emit_span(
        trace_id=..., span_id=..., span_name=...,
        resource_attributes={"service.namespace": str(ctx.app_id), ...},
        sink=sink,
    )
    ctx.telemetry.emit_log(..., sink=sink)     # raises today; see §5
    ctx.telemetry.emit_event(..., sink=sink)   # raises today; see §5
```

Why single-path for writers: the write payload is nearly identical across sinks; `ctx.telemetry` handles the wire diff. Two implementations would duplicate the whole business loop for zero benefit.

### 2.2 Readers — the dispatcher pattern (locked 2026-08-27)

For functions that query OTLP data. Two implementation functions in the same file, one dispatcher.

```python
def handler(params):
    if params.get("telemetrySink") == "oliverdb":
        return _handler_oliverdb(params)
    return _handler_postgres(params)


def _handler_postgres(params):
    """Existing code, moved here unchanged. Uses ctx.data.query('OtlpSpan',
    filters=[...]) and joins to Execution / entities in Python."""
    ...


def _handler_oliverdb(params):
    """SQL-first implementation. Uses ctx.telemetry.query('SELECT ...') for
    the span reads. Joins to Execution and Postgres entities still hit
    ctx.data.query('Execution', ...) and merge in Python -- OliverDB has no
    joins today (see oliverdb_improvements §2.2). Raises AOSException if the
    invocation has no analytics-DB config bound."""
    ...
```

**Naming convention (locked):** `_handler_postgres` / `_handler_oliverdb`. No file suffixes; both live in the same `.py`.

**Return shape parity is a hard requirement.** Both implementations return the same fields, same types, same shape. Callers never branch on sink. A dispatcher rewrite is not "done" until an A/B invocation (same params, `telemetrySink=postgres` vs `oliverdb`) returns byte-equivalent output for a fixed input dataset.

**Why not one function that branches internally.** The query languages differ (filter-dict vs. SQL). Abstracting over both hides OliverDB's real value (`percentile_cont`, `time_bucket`, attrs-key group-by, cube-served rollups). Two clean implementations read better than one function full of `if sink == …` branches.

**Why not sibling files** (`discover_agent_inventory_pg.py` + `discover_agent_inventory_oliverdb.py`). Duplicates the params contract in the routing layer, complicates ownership, and makes A/B testing awkward (two functions to invoke). The dispatcher shape keeps one endpoint, one URL, one caller signature.

---

## 3. Function inventory

Grouped by treatment. Numbers are file counts in `.trillo/568/functions/`.

### 3.1 Writers — Slice E treatment (4 total)

| Function | Status | Notes |
|---|---|---|
| `generate_live_telemetry.py` | ✅ Slice E shipped | Simulator; the flagship writer. |
| `process_execution_telemetry.py` | ✅ Slice E shipped | Execution-created trigger. |
| `backfill_scenario_traces.py` | **TODO — Slice E'** | Writes 3× (span + log + event) per Execution; also updates `Execution.traceId`. |
| `backfill_span_exceptions.py` | **TODO — Slice E'** | Reads OtlpLog, writes OtlpEvent projections. Also affected by dispatcher rules for the read side (see §3.5). |

### 3.2 Readers — Dispatcher pattern (5 in scope for the compact plan)

| Function | What it reads | OliverDB query hook |
|---|---|---|
| `calculate_execution_cost_and_tokens.py` | OtlpSpan per-execution (model derivation from LLM spans) | `WHERE trace_id = ? AND span_name LIKE 'llm.%'` |
| `reconcile_applications.py` | OtlpSpan resource attrs (application_id, service.namespace) | `GROUP BY attrs.application_id` |
| `validate_dataset.py` | OtlpSpan validation checks | `SELECT COUNT(*) FILTER (WHERE …)` variants |
| `reconcile_agent_dependencies.py` | OtlpSpan for tool/model refs | Filter by `span_kind='CLIENT'`, group by `attrs.tool`, `attrs.gen_ai.request.model` |
| `discover_agent_inventory.py` | OtlpSpan resource attrs bottom-up | `GROUP BY attrs.agent_id, attrs.service.namespace` — flagship rewrite |

Ordering is deliberate; see §6.

### 3.3 Readers — Dispatcher DEFERRED until OliverDB ships logs/events tables (2)

| Function | Deferred because |
|---|---|
| `get_correlated_logs_and_events.py` | Reads OtlpLog + OtlpEvent needle-lookup by trace_id / span_id. OliverDB has no logs/events table yet (`oliverdb_improvements §4.1/§4.2`). Rewrite when those land. |
| `get_execution_details.py` | Reads OtlpLog + OtlpEvent alongside the Execution row. Same story. |

### 3.4 Postgres-only (not migrable) (1)

| Function | Reason |
|---|---|
| `rename_model_references.py` | Bulk `ctx.data.update` on OtlpSpan rows to rewrite `attrs.model` after a model rename. OliverDB is append-only columnar — no in-place UPDATE. Behavior needed for Postgres retention windows only; when the OliverDB retention window ages the old model refs out, this becomes a no-op there. Document the constraint in the docstring; do not add a dispatcher. |

### 3.5 Reader/writer hybrid — needs mixed treatment (1)

| Function | Treatment |
|---|---|
| `backfill_span_exceptions.py` | Reads OtlpLog (Postgres always, until §4.1 lands) + queries OtlpEvent to dedup. Writes OtlpEvent. Slice E' writer treatment covers the write; the read side stays Postgres until §4.1. |

### 3.6 Stays Postgres — all others (~43)

All functions below query `Execution` / `MetricRollup` / entity tables. They already work today; they don't touch OTLP tables directly, or only through Execution's first-class token/cost/status/duration columns. **No changes proposed.**

**Aggregate sweepers (16):**
`aggregate_costs_and_tokens`, `aggregate_executive_health`, `analyze_latency`, `analyze_performance_regression`, `sweep_reliability_health`, `sweep_token_efficiency`, `sweep_metadata_completeness`, `sweep_governance_audit`, `detect_behavioral_drift`, `cluster_failures`, `forecast_costs`, `alert_rule_evaluator`, `rollup_execution_daily`, `run_sweeper_pass`, `reconcile_agent_inventory`, `reconcile_derived_state`.

**Dashboard / HTTP read-only (11):**
`get_executive_health_summary`, `get_health_status`, `get_location_status`, `get_top_platform_findings`, `get_top_token_consumers`, `get_impacted_agent_findings`, `get_agent_performance_baseline`, `get_failure_cluster_statistics`, `compare_agent_versions`, `list_versioned_agents`, `get_agent_dependency_tree`, `get_dependency_topology`.

**Entity CRUD / notification / seeds (16):**
`seed_fleet_v2`, `seed_alerting`, `seed_demo_scenarios`, `seed_drift_scenario`, `seed_model_pricing`, `seed_governance_policies`, `seed_health_policies`, `notify_alert`, `alert_dispatcher`, `set_alert_suppression`, `update_alert_status`, `update_governance_policy_action`, `restore_agent_purpose`, `prune_rule_channels`, `backfill_activity_history`, `backfill_graph_json`, `backfill_instance_address`, `write_investigation_report`, `generate_audit_evidence_package`.

---

## 4. Non-goals for this pass

- **No migration of aggregate sweepers to OliverDB.** Execution stays the aggregation source. Revisit after the OliverDB team ships the ingestion plugin API (see §7).
- **No removal of the Postgres OTLP tables.** They remain the default sink and the fallback for anything not yet migrated.
- **No metrics-signal work.** OliverDB may not even ship a metrics table; scope out until confirmed.
- **No JOIN or CTE-dependent rewrites yet** (per `oliverdb_improvements §2`). Where a rewrite needs a join, we merge in Python.
- **No shape-drift.** Any dispatcher rewrite must return byte-equivalent output for the same input across both sinks — verified per-function before the commit.

---

## 5. Toolkit prerequisites

- `ctx.telemetry.emit_span(sink=…)` ✅ shipped Slice E.
- `ctx.telemetry.emit_log(sink=…)` ✅ shipped this session — raises `AOSException` on `sink="oliverdb"` until `oliverdb_improvements §4.1` lands. Kwarg is in place so caller signatures don't shift when the table ships.
- `ctx.telemetry.emit_event(sink=…)` ✅ shipped this session — same story for `§4.2`.
- `ctx.telemetry.query(sql)` ✅ shipped Slice D. Raises when no analytics-DB config is bound.

The 5 dispatcher rewrites in §3.2 can start today with this surface.

---

## 6. Slice ordering

### 6.1 Slice E' — Complete the writers  *(next up)*

Two files, mechanical:
- `backfill_scenario_traces.py` — sink kwarg on the 3 emit calls per trace, producer discipline (`service.namespace = str(ctx.app_id)`, `attrs.trillo.execution_id`, `gen_ai.*` on any LLM span).
- `backfill_span_exceptions.py` — sink kwarg on the 1 event emit. Read side stays Postgres until §4.1 ships.

Estimate: 2 small commits, one hour.

### 6.2 Slice F' — Reader dispatcher rewrites, in risk order

Each rewrite: move existing code into `_handler_postgres` verbatim, write `_handler_oliverdb` from scratch, add dispatcher. Both paths must return the same shape on the same input. Verified by an A/B invocation before the commit.

Order chosen to build muscle on small ones and land the flagship last:

1. **`calculate_execution_cost_and_tokens.py`** — smallest. One span query, per-execution derivation. Good learning function for the pattern.
2. **`reconcile_applications.py`** — small. OliverDB path: `SELECT DISTINCT attrs.application_id …`; Postgres path stays.
3. **`validate_dataset.py`** — validation. Any discrepancy between the two paths is a real dataset issue, not a bug — surface loudly.
4. **`reconcile_agent_dependencies.py`** — larger. Derives Dependency, Model, Tool entities. Postgres writes for the derived entities are the same in both paths.
5. **`discover_agent_inventory.py`** — flagship. Biggest reader of raw OTLP. Do this last, benefiting from the previous four's learnings.

Estimate: 5 commits, one per function, ~half a day each including A/B verification.

### 6.3 Slice G' — Deferred until OliverDB ships §4.1 / §4.2

- `get_correlated_logs_and_events.py` — dispatcher rewrite once OliverDB has logs + events tables. Estimate the smaller sibling; this reads across both.
- `get_execution_details.py` — dispatcher rewrite once logs/events land.

Both are read-only needle lookups; deferring costs nothing in the meantime.

### 6.4 Slice H' — Documented Postgres-only  *(one-line change)*

- `rename_model_references.py` — add a note in the docstring explaining why this function will never route to OliverDB (append-only columnar; no UPDATE). Ten-minute change; do at any time.

---

## 7. Denormalization at ingestion — future consideration

When the OliverDB team ships the ingestion-time plugin API (per `OliverDB-Otel-Mapping-Requirements.md` — the Rust in-process plugin option), a class of Postgres-only sweepers becomes migration candidates:

- **Cost derivation at ingest.** `attrs.trillo.cost_usd = tokens × pricing_lookup(model)` computed per span at write time. `forecast_costs` becomes a straight OliverDB SQL sum instead of a Postgres query + Python join to `ModelPricing`.
- **Eval pass-rate at ingest.** `attrs.trillo.eval_pass = evalScore >= threshold`. `sweep_governance_audit` reads it as a boolean group-by instead of scoring in-flight.
- **A/B version bucketing at ingest.** `attrs.trillo.canary_bucket = A|B` decided by the plugin from the seed roster. `compare_agent_versions` becomes a filter instead of a version-string join.
- **Trace-level aggregates at ingest.** Total trace duration and error-count-per-trace materialized on the root span, so `WHERE root AND total_duration_us > 30_000_000 AND error_count >= 1` is a single scan instead of the subquery we don't have today.

None of these are actionable until the plugin API is documented and running. Track this as an open question with the OliverDB team; revisit this plan when the surface is known. See also `oliverdb_improvements §1.2` (semconv pull-outs at ingest — the same mechanism, simpler case).

---

## 8. Open issues

1. **`process_execution_telemetry.py` as a trigger.** Slice E leaves the raise semantic on `sink="oliverdb"` when misconfigured. That's fine while the emitter defaults to `postgres` — a broken OliverDB slot never affects trigger runs. If a future admin flow ever pushes `telemetrySink=oliverdb` into the trigger's params, a misconfigured slot fails every Execution create. Guard with a fallback in the trigger dispatcher, or accept it? My lean: **guard**. The trigger is invisible to users; hard failures there are indistinguishable from Execution creation bugs. Add an explicit try/except around `emit_span(sink="oliverdb")` in trigger paths that logs a warn and falls back to Postgres for that turn.
2. **Reader shape parity testing.** No automated test framework yet. Should we build a small harness — invoke both handlers over a fixed seed dataset, diff outputs, fail the CI if they diverge? Or manual verification per rewrite? My lean: **manual for the first five**, harness in Slice I' if we do more.
3. **`ctx.telemetry.query` needs a read-scoped key.** Slice C injects only a write-scoped key today. The dispatcher `_handler_oliverdb` paths won't run in the pod until read keys are provisioned. Options: extend Slice C to inject both, or handle in Slice F' as the first sub-task. Blocks Slice F' — decide before starting.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-08-27 | Trillo (via Claude Code session) | Initial draft. Scope (i) locked; 5 dispatcher targets; 2 remaining writers; documented Postgres-only for one function. Trigger-in-prod concern flagged. |
