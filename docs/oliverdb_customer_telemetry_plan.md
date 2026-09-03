# OliverDB — Customer Telemetry Alignment Plan

**Document Version:** 0.1 (draft, pre-execution)
**Purpose:** enumerate every change needed to align the OliverDB / `ctx.telemetry` stack for customer-emitted telemetry from ADK / LangChain / LangGraph / CrewAI, plus the outbound TODO items for the OliverDB team and the Trillo platform team.
**Companion to:**
- `oliverdb_entities/OtlpTelemetry.json` — the target schema for OliverDB
- `oliverdb_ingest_normalization_note.md` — the hand-to-OliverDB brief
- `ingest_to_oliverdb_from_agent_frameworks.md` — the customer-facing framework recipes
- `app_team_oliverdb_guide.md` — the Trillo-app developer guide

## 1. Design decisions locked in this pass

1. **`executionId` becomes nullable.** Customer telemetry has no equivalent; forcing it as required would reject conforming customer records.
2. **Ingest-plugin derivation precedence for `executionId`** (first hit wins):
   1. Producer-explicit column value (Trillo `ctx.telemetry` emit_span with `execution_id=…`).
   2. `attrs.trillo.execution_id` (a Trillo-adjacent producer wrote it into attrs but not the column).
   3. `attrs.session.id` (OTel semconv v1.30+).
   4. `traceId` (span/event/log has one; often missing on metrics).
   5. `NULL`.
3. **`sessionId` is a new flat column** with its own cross-framework derivation precedence:
   1. `attrs.session.id` (OTel canonical).
   2. `attrs.langsmith.session_id`.
   3. `attrs.adk.session.id`.
   4. `attrs.langgraph.thread_id`.
   5. `attrs.crewai.session.id`.
   6. `NULL`.
4. **`userId` is a new flat column** — try `attrs.enduser.id`, `attrs.user.id`, `attrs.langsmith.user_id`, then `NULL`.
5. **`genaiOperationName` is a new flat column** — one of OTel semconv v1.28+ values (`chat`, `text_completion`, `embeddings`, `create_agent`, `execute_tool`, `generate_content`). No Trillo extensions on this list.
6. **Trillo attribute names are not a driving factor** for the semconv promotion — we're aligning to OTel, not codifying Trillo-flavored keys into the ingest plugin.
7. **Postgres side is untouched.** All existing `entities/Otlp*.json` schemas stay at their pre-b36a297 shape. Existing Trillo sweepers keep working.

## 2. Phase 1 — schema + OliverDB team note

Two file edits before anything else.

### 2.1  `oliverdb_entities/OtlpTelemetry.json`

Changes:

| Field | Change |
|---|---|
| `executionId` | drop `required: true`; update description to name the four-level derivation precedence |
| `sessionId` | **new**, string, indexed. Description covers cross-framework key list |
| `userId` | **new**, string, indexed |
| `genaiOperationName` | **new**, string, indexed. Enum-typed to OTel semconv v1.28+ values |

Result: 72 columns → 75 columns; `executionId` optional; three new flat semconv promotions.

### 2.2  `oliverdb_ingest_normalization_note.md`

Add three new sections:

- **§9 Customer telemetry framing** — one-paragraph statement that the plugin must gracefully handle telemetry from customer agents (ADK/LangChain/LangGraph/CrewAI) that don't emit Trillo-specific keys. The plugin is exactly what closes that gap.
- **§10 `executionId` derivation precedence** — the four-level list from §1.2 above, with a note that metric rows will often land NULL and that's fine.
- **§11 `sessionId` + `userId` cross-framework normalization** — the ordered key lists, with the semantic argument for why OTel canonical wins ties.

Also update:
- The 15-attr promotion list → 18-attr list (+ `session.id`, `enduser.id`, `gen_ai.operation.name`).
- The "Five questions" section: add one more asking whether the plugin's derivation rules can be composed (attr-key promotion + derivation-from-attrs both firing during the same batch pass).

**Estimate:** 30 minutes.

## 3. Phase 2 — Trillo code and docs

Follow-up commits after Phase 1 lands.

### 3.1  Code changes — emitters (4 files)

Only necessary work is what's needed for the simulator + trigger to emit `session.id` so their traces group correctly in the new dashboard queries. Everything else the emitter does is unchanged.

| File | Change | Why |
|---|---|---|
| `generate_live_telemetry.py` | Set `attributes["session.id"]` on every emitted span using the existing `Execution.sessionId` (already generated as `sess-XXXX`) | Simulator traces will collapse per-session in the waterfall view; matches customer-telemetry semantics |
| `process_execution_telemetry.py` | Same — pull `execution.get("sessionId")` and set `attrs.session.id` | Consistency with the simulator |
| `backfill_scenario_traces.py` | Same — set `session.id` from the demo-seed Execution | Backfilled traces get session correlation |
| `backfill_span_exceptions.py` | No change | Only creates exception events; inherits session from parent span (or lands NULL, which is fine) |

**Estimate:** 20 minutes across all four files.

### 3.2  Code changes — readers (up to 4 files, optional)

These reader dispatchers today query OliverDB via `attrs.trillo.execution_id` in SQL. Once the plugin populates the typed `executionId` column, they can switch to filtering by that column directly — faster (typed vs attrs-key), and cleaner SQL. Optional; queries continue to work either way.

| File | Change | Priority |
|---|---|---|
| `calculate_execution_cost_and_tokens.py` | `WHERE attrs.trillo.execution_id = 'X'` → `WHERE executionId = 'X'` (in `_handler_oliverdb`) | Medium |
| `reconcile_applications.py` | Same substitution in `_ra_from_span_oliverdb` | Medium |
| `reconcile_agent_dependencies.py` | Change ts-window filter target from `attrs.trillo.execution_id IN (…)` to `executionId IN (…)` | Medium |
| `discover_agent_inventory.py` | The big SELECT reads `attrs.trillo.execution_id` — swap to `executionId` | Medium |
| `validate_dataset.py` | No change (uses `span_name` LIKE, not executionId) | — |

**Estimate:** 30 minutes across four files.

**Blocker:** only meaningful after the OliverDB plugin ships and starts populating the `executionId` typed column. Sequence: Trillo dual-writes `executionId` today (via `emit_span(execution_id=…)`), reader switch after plugin is live.

### 3.3  Doc updates (2 files)

**`ingest_to_oliverdb_from_agent_frameworks.md`** — add:

- New section "Session and user correlation" — recommend customers set `attrs.session.id` per conversation + `attrs.enduser.id` per authenticated user.
- Explicit statement that `executionId` is Trillo-only; customers correlate via `traceId` + `sessionId`; the plugin's derivation fills `executionId` from them.
- Per-framework hints on where session lives: ADK `adk.session.id`; LangChain `session.id`; LangGraph `langgraph.thread_id`; CrewAI manual.

**`app_team_oliverdb_guide.md`** (the living reference; §25 in aos-docs re-syncs from this later):

- Small clarification that `ctx.telemetry.emit_span(execution_id=…)` still explicitly sets the value; the plugin's derivation kicks in only when the producer doesn't declare an explicit executionId (i.e., customer telemetry).
- New "Correlation across the two data sources" callout noting that Trillo-emitted spans have `executionId` set explicitly; customer-emitted spans get executionId derived from session.id or trace_id at ingest.

**Estimate:** 30 minutes across both docs.

## 4. TODO — for the Trillo platform team (Anil + engineers)

Ordered by dependency.

- [ ] **T-1** — Execute Phase 1 §2.1 (schema update). Commit + push. *Blocker for T-2.*
- [ ] **T-2** — Execute Phase 1 §2.2 (OliverDB team note update). Commit + push.
- [ ] **T-3** — Share the updated `oliverdb_ingest_normalization_note.md` with the OliverDB team. Ask for confirmation on the derivation-composition question (§2.2 last bullet), and estimated plugin timeline.
- [ ] **T-4** — Execute Phase 2 §3.1 (four emitter file updates). Commit + push.
- [ ] **T-5** — Execute Phase 2 §3.3 (both doc updates). Commit + push.
- [ ] **T-6** — Update the `app_team_oliverdb_guide.md`'s living-reference tag to note that this pass has aligned the developer-facing behavior with what the plugin will implement.
- [ ] **T-7** *(after OliverDB plugin ships)* — Execute Phase 2 §3.2 (reader-dispatcher optimization to use typed `executionId` column). Commit + push per file. Verify shape parity on each.
- [ ] **T-8** *(after OliverDB plugin ships)* — Update `aos-docs §25` from the living `app_team_oliverdb_guide.md`.

## 5. TODO — for the OliverDB team

To share with them alongside the updated `oliverdb_ingest_normalization_note.md`.

- [ ] **O-1** — Confirm whether the ingest-path plugin surface exists today or is a design item. If design, estimate landing quarter.
- [ ] **O-2** — Confirm whether promotion rules can be **declarative config** vs Rust plugin code. Trillo prefers declarative rules for iterability without a release cycle.
- [ ] **O-3** — Confirm whether promotion rules can also do the derivation described here (multi-source with first-hit-wins fallback), not just simple 1-to-1 key-to-column extraction. If not, we'll need a Rust plugin for the derivation logic even if config-driven promotion suffices for the simple case.
- [ ] **O-4** — Confirm behavior on type mismatch (producer emits `gen_ai.request.model` as a number instead of string). Trillo's expected behavior: land the record with the column NULL and a per-record annotation; don't reject.
- [ ] **O-5** — Confirm whether promoted columns need to be declared at schema-creation time or can be added incrementally over the tenant's lifetime.
- [ ] **O-6** — Confirm per-column storage-cost visibility (dictionary size, RLE ratio, distinct-value count) so Trillo can tune the promoted set over time.
- [ ] **O-7** — Test harness or preview mode for promotion rules before deploying to a prod tenant.

## 6. TODO — for the UI team

- [ ] **U-1** — Once the Designer's AppConfig editor ships the two `analyticsDb*` fields (already tracked in `analytics_db_appconfig_editor_ui_handoff.md`), no new work here from this pass. This item is a reminder-of-scope, not a new ask.

## 7. TODO — for the customer-facing docs team

- [ ] **D-1** — Absorb the `session.id` + `enduser.id` recommendations from the updated `ingest_to_oliverdb_from_agent_frameworks.md` into any external onboarding material for the four frameworks.
- [ ] **D-2** — When aos-docs `§25` gets its re-sync (T-8), update any customer-facing quickstart that references `executionId` to note that it's Trillo-only and customers get it via plugin derivation.

## 8. Open questions

1. **Timeline pressure on the plugin.** If the OliverDB plugin timeline is >6 weeks, do we want to invest more in the Trillo-side simulator changes (Phase 2 §3.1) to fully compensate, or accept the intermediate state (Trillo-emitted data has session.id, customer-emitted data has whatever the framework produces + plugin populates the typed columns lazily)?
2. **Metrics with NULL `executionId`.** Confirmed OK (§1.2's decision), but do we want a sweeper that samples-and-annotates metric rows with the executionId of their exemplar's span? Optional exemplar-based backfill. Deferred; revisit if metric-to-conversation correlation becomes a common query.
3. **`sessionId` derivation from producer-side alternative keys.** Some customers may have a `chain.id` or `run.id` concept from their own producer wrapper that maps to "session" semantically. Do we want to expose a per-tenant extension list? Deferred.

## 9. Estimated total effort

- Phase 1 (schema + OliverDB note): ~30 minutes.
- Phase 2 §3.1 (emitters): ~20 minutes.
- Phase 2 §3.2 (readers, optional, deferred): ~30 minutes.
- Phase 2 §3.3 (docs): ~30 minutes.
- **Trillo total to done-except-plugin-blockers**: ~80 minutes.

Total wall-clock, if execution runs sequentially without OliverDB-team blockers: half a day.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-09-03 | Trillo (via Claude Code session) | Initial plan. Locks the derivation precedence for `executionId` and `sessionId`; enumerates schema changes, code changes, and cross-team TODOs. Awaiting approval before Phase 1 execution. |
