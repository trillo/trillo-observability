# Enterprise AI Agent Observability & Analytics
## Telemetry Simulator — Requirements

**Document Version:** 0.3 (draft)
**Companion to:** POC Requirements v1.5 + Requirements Addendum (decision log)
**Platform:** Trillo AOS (observability application); simulated source = generic
framework agents (LangChain/other) on Google Cloud / edge — see Addendum AD-003.

---

## 1. Purpose

In production, telemetry is emitted by the agents themselves. For the POC, a
**Telemetry Simulator** generates realistic, OTel-shaped telemetry for **N
agents** so the observability application can be demonstrated end-to-end without
live agents.

The simulator must produce data that is:
- **OTel-conformant** — traces/spans, metrics, events, logs matching the PRD
  schema (§11.3) and OTel GenAI semantic conventions.
- **Internally consistent** across all seven scenarios (PRD §2.4, §11.3.5) — the
  same operational story reconciles on every screen.
- **Realistic** — believable latency/cost/token/error distributions and 30–90
  days of history with diurnal/weekly patterns, growth, and anomalies.
- **Pipeline-faithful** — the simulator emits **raw** telemetry; the platform's
  own sweepers/processors build rollups, findings, baselines, and topology
  (AD-006), so the POC exercises the real aggregation path. (A backfill mode may
  also seed deep-history rollups for dashboard responsiveness.)

## 2. Scope of generated data — telemetry only; inventory is DERIVED (AD-010)

The simulator **emits telemetry**; it does **not** pre-populate inventory tables.
The inventory entities are **derived by sweepers** from the telemetry (the
production-faithful OBSERVED discovery path).

1. **Telemetry signals (the simulator's output):** traces/spans, metrics, events,
   logs — each carrying the **resource + span attributes** needed to reconstruct
   inventory (see §3 / §12): `agent_id`, `agent_name`, `application_id`,
   `owner_team`, `cost_center`, `business_unit`, `governance_class`,
   `agent_version`, `location_id`, `service_instance_id`, cluster/pod/project or
   edge id, `environment` (**resource** attrs); `request_model`/provider,
   `tool_name`, `dependent_system`, retrieval target (**span** attrs).
2. **`model_pricing`** — the one **reference table** the simulator seeds as dummy
   data (pricing is not telemetry).
3. **DERIVED by platform sweepers (NOT the simulator):** `applications`, `agents`
   (logical), `agent_instances`, `models`, `tools`, `external_systems`, vector
   stores, `agent_dependencies` (OBSERVED), plus `metric_rollups`,
   `platform_findings`, `analysis_baselines`, `ai_analyses`,
   `optimization_recommendations`.
4. The **world config** (§11) still declares the intended agents/models/tools/
   dependencies — but only to **drive what the telemetry emits**, not as
   pre-written rows.
5. To demonstrate **unowned / metadata-incomplete** agents, the simulator
   deliberately **omits** some resource attrs (owner/purpose/application) for
   selected agents, so the Inventory/Metadata sweeper flags them.
6. The simulator MAY optionally pre-seed historical rollups for dashboard
   responsiveness.

## 3. Agent & deployment model (per this request)

Each generated record carries full agent identity + deployment context.

- **Identity (AD-001):** the agent **emits** `agent_id` (stable, logical),
  `agent_name`, `agent_version`, `application_id`. `agent_id` is stable across
  versions/instances; `agent_version` is a separate dimension.
- **Deployment target — one of:**
  - **GKE agent-sandbox pod:** `gcp_project_id`, `cluster_id`, `namespace_name`,
    `pod_name`, `service_instance_id`, sandbox indicator, `environment`.
  - **Edge:** edge site/device id, `service_instance_id`, `environment`.
- **Location / region (published by the agent):** `location_id` drawn from a
  configurable region set — **East, West, South, North, Central** (extensible,
  e.g. per-country/metro). The exec dashboard shows thousands of active locations
  (§10.3), so the simulator must spread instances across many locations.
- **Instances:** thousands of `agent_instances` per §11.5; a logical agent maps to
  many instances across locations/versions/environments, each with first/last
  seen.

### 3.1 Agent vs Agent Instance — the triage axis (frames this document)

- **Logical Agent** = the code/definition (prompt, tools, model, logic), identical
  across deployments. **Agent Instance** = one deployment of it at one location
  (GKE sandbox pod / edge). Executions and traces happen on **instances**, so
  every trace is intrinsically **instance- and location-scoped**.
- This split is the platform's **root-cause axis**, and the simulator must make it
  demonstrable by generating **two distinct failure modes**:
  1. **Deployment / environment failures** — concentrated in **one instance /
     location / cluster** (bad pod, region-local dependency outage, config drift).
     Blast radius = narrow.
  2. **Code / logic failures** — the same failure signature **spread across many
     instances** of one logical agent, across locations, typically pinned to an
     **`agent_version`** boundary. Blast radius = wide.
- Therefore each seeded failure cluster carries a controllable **spread** (distinct
  instances / locations / versions affected) so the platform can classify
  code-vs-deployment ("many instances of one type failing ⇒ likely a code issue;
  one instance failing ⇒ likely a deployment issue"). Seed **at least one of each
  mode** in the story.
- **Trace scoping consequence:** the simulator should distribute executions/traces
  across instances/locations so the app can show aggregates at the logical-agent
  level and reserve full trace lists for the instance level (per the POC app
  design; per-logical-agent trace firehose is intentionally avoided).

## 4. Trace / span generation

Generate span trees per the Addendum model (AD-004): **Session → 1..N traces →
tree of spans**, spanning all categories, not just agents.

- **IDs:** `trace_id` (128-bit), `span_id` (64-bit), `parent_span_id`;
  `execution_id` on the root; `session_id` groups multiple traces (multi-turn
  sessions + background traces).
- **Span categories/kinds:** `AGENT` (root + nested sub-agents), `MODEL` (LLM
  call), `TOOL`, `RETRIEVAL` (vector store), plus downstream `HTTP` / `DATABASE`
  under tools, and `FUNCTION`. Set `span_kind` (SERVER/CLIENT/INTERNAL) and
  `span_category` consistently.
- **Structure:** nested agents produce child `AGENT` spans; tools carry
  `tool_name` + `dependent_system`; a tool's downstream call is a child HTTP/DB
  span. Each agent's tool/model set is drawn from its configured dependency set —
  **and deliberately varies across traces** (an agent may use `{X,Y}` in one
  trace, `{X,Z}` in another) so the union-over-time topology (AD-006) is
  exercised.
- **Distributed traces:** a configurable fraction of sub-agent calls cross a
  service boundary (separate trace + span link / parent ref) to exercise the
  distributed case (AD-004 / §11.3.5 rule 2).
- **Timing:** per-span `start_time`/`end_time`/`duration_ms` with realistic,
  category-specific latency distributions (model vs tool vs retrieval vs
  orchestration), producing coherent P50/P90/P95/P99. Include **anomalous
  windows** (latency spikes) that the Latency scenario (§6) can surface.
- **Tokens & cost:** per `MODEL` span — `input_tokens`, `output_tokens`,
  `cached_tokens`, `reasoning_tokens`, `total_tokens`; `estimated_cost_usd`
  computed from effective-dated `model_pricing`; `request_model`/`response_model`.
- **Status & failure:** OK/ERROR with `status_message`; inject **failure
  clusters** (e.g., one tool → one system returns HTTP 504 across many locations
  in a time window, §5.3) so reliability + impacted-systems + executive views all
  reconcile.
- **Prompt/completion:** synthetic-but-plausible `prompt_text`/`completion_text`
  (subject to masking policy), including **token-waste patterns** for the
  Optimization scenario (bloated prompts, repeated context across turns,
  retrieved-but-unused context, high input:output ratio, oversized model for a
  trivial task).

## 5. Metrics generation

Emit OTel/GenAI + POC-derived metrics (§11.3.2), dimensioned by application /
agent / agent_version / model / tool / owner / cost_center / environment /
location / time:
- `gen_ai.client.token.usage`, `gen_ai.client.operation.duration`, execution
  count, success/error count, cost, latency percentiles, adoption/volume,
  governance pass/fail.
- Support both **raw metric points** and **pre-aggregated windowed points**
  (`1m/5m/1h/1d`) for responsive dashboards; aggregates must **reconcile** with
  the underlying spans (§11.3.5 rule 8).

## 6. Events generation

Emit `otlp_events` (§11.3.3) correlated by `trace_id`/`span_id`/`execution_id`:
- **Exceptions** paired with every error span (+ correlated logs).
- **Evaluations:** toxicity / hallucination / PII-leak with `eval_score` +
  `eval_label` (PASS/FAIL).
- **Policy decisions:** `policy_id`/`policy_version` + `ALLOW|WARN|REDACT|
  REQUIRE_APPROVAL|BLOCK`, so the Governance scenario (§9) has real material,
  including a policy that flips Warn→Block.

## 7. Logs generation

Emit `otlp_logs` (§11.3.4) with `severity_text` DEBUG..FATAL, correlated to
traces/spans/executions. Failure scenarios include coherent diagnostic logs
(connection retries, timeouts, stack traces in `log_attributes`) so the SRE
root-cause flow (§13.5) has grounded evidence.

## 8. Consistency & scenario-seeding requirements

The simulator must not emit random noise — it seeds the **specific narratives**
the demo needs and keeps them consistent across screens:
- Enforce all §11.3.5 correlation/data-quality rules (root span per execution;
  valid parents; failure ⇒ error span + exception event + logs; model spans carry
  token/cost; tool spans carry `tool_name`+`dependent_system`; consistent
  dimensions; rollups reconcile).
- Seed the storyline (§14): a recent **reliability degradation**, a **cost
  increase**, **token-waste** examples, **governance** pass/fail cases, and
  **unowned / metadata-incomplete** agents (§4, §13.3).
- Seed enough clean history for **baselines/regression** (§13.3) so
  current-vs-baseline comparisons are meaningful.

## 9. Volume, time & determinism controls

- **Configurable scale:** N agents (default 15–25, §11.5), instances per agent,
  locations, executions per unit time, history depth (30–90 days), error rate,
  latency/token/cost distributions, anomaly windows.
- **Two run modes:**
  1. **Backfill** — bulk-generate historical data (fast load).
  2. **Live / streaming** — continuous emission to exercise near-real-time
     processors and live dashboards.
- **Deterministic seeding** — a random seed makes datasets reproducible;
  re-running yields the same story. Live mode appends without breaking history.
- **Simulated clock** — backdate historical generation and optionally compress
  time; all timestamps derive from the sim clock, not wall-clock.

## 10. Output / delivery

- **Sink-abstracted** (AD-006): write via the **Trillo AOS ingest API** and/or
  direct rows into the POC store. For the POC that is PostgreSQL; the same emitter
  targets the columnar store in production. Prefer **OTLP** payloads (what real
  LangChain/OpenInference/OpenLLMetry agents emit) so the ingest path is
  production-faithful, with a direct-insert fast path for bulk backfill.
- **Schema conformance:** every record maps to `otlp_spans` / `otlp_metrics` /
  `otlp_events` / `otlp_logs` (+ fixtures), including the denormalized business
  dimensions and the `raw_attributes`/`*_json` catch-alls.

## 11. Configuration (the simulator "world spec")

A declarative config (YAML/JSON) defines the world and generation parameters:
- **Fixtures:** applications, agents (with owner/purpose/cost-center/governance
  class), models (provider/tier/pricing), tools, external systems, vector stores;
  per-agent dependency sets and version lineage.
- **Deployment topology:** GKE (projects/clusters/namespaces) and edge sites;
  region/location list; instance counts + spread.
- **Generation params:** rates, distributions, error/anomaly injection, scenario
  seeds, history depth, seed, run mode.

## 12. Other useful information to generate (added per request)

- **User & session context:** `user_id`, `session_id`; multi-turn sessions (one
  session = several traces) and some background/scheduled traces.
- **Cost model realism:** effective-dated `model_pricing`; cached/reasoning token
  pricing; deliberate **cost-growth** trends over the history window; a couple of
  expensive-model-for-simple-task cases for Optimization.
- **Metadata quality:** deliberately create **unowned** and **metadata-incomplete**
  agents, and **stale** instances (old `last_seen`) for the Inventory/Metadata
  scenarios.
- **Version rollouts:** multiple `agent_version`s per agent with a **regression**
  introduced at a version boundary (latency/error up) for Performance Regression
  (§13.3).
- **A2A / dependency variety:** agent→agent edges, shared sub-agents across
  parents, and tools that fan out to multiple systems, so topology (AD-002/005)
  is rich.
- **Governance material:** PII/toxicity samples, redaction/masking flags,
  versioned policies, and one administrative policy change (Warn→Block) with an
  audit record.
- **Guardrail/eval distributions:** mostly PASS with a small, believable FAIL
  tail so the Guardrail Pass Rate (§10.3) reads ~99.9%.
- **Correlation completeness:** propagate `application_id`, `agent_id`,
  `agent_version`, `location_id`, `user_id`, `session_id` consistently across
  spans/events/logs/metrics (§11.3.5 rule 7).

## 13. Acceptance criteria

- All four signal types generated and schema-conformant; every execution has a
  root span with matching `execution_id`/`trace_id`.
- The seven-scenario story reconciles across screens (an injected inventory-tool
  failure shows in Inventory, Reliability, Latency, and the Executive dashboard).
- Dashboard aggregates reconcile with the underlying spans/metrics for any window.
- Sweepers, run against generated data, produce sensible rollups/findings/
  topology without simulator-side pre-computation (pipeline proven).
- Reproducible from a seed; supports both backfill and live modes; sink-abstracted
  (Postgres POC, columnar prod).

## 14. Open items

- Exact target rates for the POC (executions/min) and total history volume.
- Whether to pre-seed historical rollups (responsiveness) in addition to raw.
- Region taxonomy beyond East/West/South/North/Central (per-country/metro?).
- HTTP/DATABASE span labeling: extend `span_category` vs. rely on `span_kind` +
  attributes (flag from Addendum; decide at Scenario 3).
