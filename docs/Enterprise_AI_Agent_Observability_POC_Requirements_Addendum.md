# Enterprise AI Agent Observability & Analytics
## Requirements Addendum (Decision Log)

**Addendum Version:** 0.14 (in progress)
**Base Document:** Enterprise AI Agent Observability & Analytics — Proof of Concept
Requirements and Demonstration Specification, **v1.5**
**Platform:** Trillo AOS
**Status:** Living document — updated during requirements Q&A.

---

## 1. Purpose

This addendum records decisions made during requirements Q&A on the base PRD
(v1.5). It is the authoritative delta: the base document remains unchanged; each
decision below **adds**, **clarifies**, or **amends** a specific part of it.

> **Scope (see AD-003):** The **observed agents** are Wendy's full-stack
> application on Google Cloud, built with **generic agent frameworks (LangChain
> or other)** — NOT Trillo AOS agents. **Trillo AOS** is the platform on which
> the **observability application** is built (§12), i.e. the analytics/consumer
> side. Telemetry originates from Wendy's framework-instrumented agents emitting
> OTel. Do not conflate this with instrumenting Trillo AOS's own agent runtime
> (a separate initiative).

Each entry is self-contained so this addendum can be delivered alongside the PRD
and read as the change record.

## 2. How to read this document

- **Decision ID** — stable reference (e.g., `AD-001`).
- **Relationship** — `ADDS`, `CLARIFIES`, or `AMENDS` the referenced PRD section.
- Where a decision supersedes a prior one, the older entry is marked
  **Superseded by <ID>** and kept for traceability.

---

## 3. Decision Log

### AD-001 — Agent identity vs. version in telemetry

- **Date:** 2026-08-12
- **Area / Topic:** Agent identity, versioning, telemetry dimensions
- **Relationship:** CLARIFIES — PRD §3.2, §3.3, §11.2.4, §11.2.5, §11.2.10, §11.3.1
- **Question:** Does telemetry carry `agent_id`? Is it unique? Does it change with
  the agent version, or do we need to add version info to the telemetry data?
- **Decision:**
  1. Telemetry carries `agent_id` (and `agent_name`) on both `agent_executions`
     and `otlp_spans`.
  2. `agent_id` is the **logical** agent identity — unique per named capability
     and **stable**; it does **not** change across versions or runtime instances.
  3. Version is a **separate dimension already in the model**: `agent_version` on
     `agent_executions`, `agent_instances`, and `otlp_spans`; `agents.current_version`
     on the logical row. Version info does **not** need to be added — it must be
     **populated** on every telemetry row.
  4. Identity levels are distinct and stack:
     `agent_id` (logical, stable) → `agent_version` (version dimension) →
     `service_instance_id` / `instance_id` (runtime deployment; many per agent) →
     `execution_id` / `trace_id` (per execution / turn).
- **Rationale:** Version-over-version analytics — latency by agent version
  (§6.2) and the Performance Regression Analyzer's current-vs-baseline by
  *agent + version* (§13.3) — require a stable `agent_id` plus a separate version
  dimension. Baking version into `agent_id` would break regression/baseline
  comparison and fragment the inventory.
- **Impact:** Every span/execution must set both `agent_id` and `agent_version`.
  Analytics group by `agent_id` and pivot by `agent_version`. Inventory (`agents`)
  holds one row per logical agent carrying `current_version`.
- **Open sub-decisions (recommendations pending confirmation):**
  - **(a) How ingestion obtains the stable `agent_id`.** **V1 decision (Accepted):
    V1 assumes the agent EMITS its `agent_id`** as a resource attribute (e.g.
    `trillo.agent.id`); ingestion trusts the emitted value. The resolve-by-name
    fallback (from `agent_name` + application + environment via the Agent
    Inventory Reconciler, §13.3) is **deferred to a later version.**
  - **(b) What defines an agent "version". Accepted (2026-08-13):** the agent
    **emits an explicit semver / build tag** as `agent_version` (its deployment/
    release version, e.g. `service.version`), consistent with (a)'s emit-first
    approach. (The earlier "AgentM content hash" idea does **not** apply — these
    are not Trillo AOS agents; see **AD-003**.)
- **Status:** **Accepted** — core model; (a) agent emits `agent_id` (V1);
  (b) agent emits semver/build tag as `agent_version`.

### AD-002 — Deriving agent topology from span data

- **Date:** 2026-08-12
- **Area / Topic:** Dependency/topology discovery, span-tree traversal
- **Relationship:** CLARIFIES — PRD §11.2.9 (`agent_dependencies`), §13.3
  (Dependency Reconciler), §11.3.5 (correlation rules)
- **Question:** Can a sweeper build an agent's topology (the components it
  connects to — sub-agents, tools, models, systems) by scanning span data? In the
  span tree, does the tree extend through agent nodes?
- **Decision:**
  1. Yes. A sweeper reconstructs topology from `otlp_spans` by walking each trace
     via `parent_span_id` — this is the **OBSERVED** `discovery_source`
     populating `agent_dependencies`.
  2. Confirmed: **the span tree extends through agent nodes** (and many
     non-agent nodes — model, tool, HTTP, DB, function; see **AD-004**). A
     nested agent yields a child agent span, and its own tools/models/sub-agents
     hang beneath it.
  3. **Attribution rule:** a component (MODEL / TOOL / RETRIEVAL / sub-agent) is
     attributed to its **nearest ancestor `agent.turn` span**, not the trace
     root. Each agent "owns" the region of the tree from its `agent.turn` down to
     (but excluding) the next `agent.turn`. This prevents mis-attributing a
     sub-agent's tools to the parent.
  4. **Edge types:** agent→model (MODEL span, `request_model`); agent→tool (TOOL
     span, `tool_name`) plus the chained **tool→system** via the same span's
     `dependent_system`; agent→vector store (RETRIEVAL span); agent→sub-agent
     (adjacent parent/child `agent.turn` spans).
  5. **Aggregate across many traces** → union of edges with `first_seen_at` /
     `last_seen_at`, `confidence` (observation frequency), `evidence_trace_id`.
- **Caveats:**
  - Coverage is **observation-limited** — a rarely-exercised dependency appears
    only once observed. Complement OBSERVED with **REGISTERED** (declared from the
    agent/AgentM definition) per the `discovery_source` enum.
  - **In-process** sub-agents (same-process framework call with context
    propagation) stay in one trace → clean parent/child. A sub-agent invoked
    across a service/network boundary (HTTP, agent-to-agent, queue) starts a
    **distributed boundary** (§11.3.5 rule 2) → needs trace-linking (span link /
    parent trace ref) to stitch the topology. Which case applies depends on how
    Wendy's agents are composed (framework-dependent).
- **Impact:** Dependency Reconciler implements the nearest-ancestor-agent walk +
  cross-trace aggregation; topology view (§4.3 Dependencies tab, §5 impacted
  systems) is driven from the aggregated `agent_dependencies`. **This walk IS the
  "Agent Execution Tree" projection (see AD-004)** — the full span tree also
  contains non-agent spans (model/tool/HTTP/DB/function) that this projection
  collapses.
- **Status:** Accepted (approach + attribution rule); refined by AD-004.

### AD-003 — Scope: the observed agents are Wendy's generic-framework agents, not Trillo AOS agents

- **Date:** 2026-08-12
- **Area / Topic:** POC scope, telemetry source, platform roles
- **Relationship:** CLARIFIES — whole PRD; esp. §3 (Core Information Model),
  §11.3 (OTel schema), §12 (Trillo AOS Role)
- **Clarification:**
  1. The agents under observation are deployed by **Wendy's on Google Cloud** as
     a full-stack application, built with **generic agent frameworks (LangChain
     or other)**. They are **not** Trillo AOS agents.
  2. **Trillo AOS** is the platform on which the **observability application**
     (this POC and its production successor) is built (§12) — the
     analytics/consumer side, not the observed agent runtime.
  3. Telemetry (`otlp_spans`/`otlp_events`/`otlp_metrics`/`otlp_logs`) originates
     from Wendy's framework-instrumented agents emitting **OpenTelemetry** (e.g.
     OTel GenAI semantic conventions / OpenInference / OpenLLMetry → OTLP).
  4. The separate design "AOS as a native OTel producer"
     (`trillo-aos/docs/agent-telemetry-otel-producer.md`) instruments Trillo
     AOS's *own* agents and is a **different initiative**. AOS agents could later
     be one more observed source, but they are **not** the subject of this PRD.
- **Impact on prior decisions:**
  - **AD-001(a):** "the agent emits `agent_id`" = a **source-side instrumentation
    requirement on Wendy's agents** (emit a stable `agent_id` resource attribute).
  - **AD-001(b):** version = the agent's **emitted deploy/release version**
    (framework/CI/`service.version`), not any AgentM content hash.
  - **AD-002:** span-based topology holds for generic OTel spans; the
    in-process-vs-distributed boundary is framework-dependent (context
    propagation vs HTTP/A2A/queue), not AOS-specific.
- **Status:** Accepted (scope clarification).

### AD-004 — Session / trace / span hierarchy; span tree is not agent-limited; two derived tree views

- **Date:** 2026-08-12
- **Area / Topic:** Trace/session modeling, span taxonomy, UI tree views
- **Relationship:** CLARIFIES / AMENDS — PRD §11.2.2 (trace/execution modeling),
  §11.2.10 (`agent_executions`), §11.3.1 (`otlp_spans`), §11.3.5; **refines AD-002**
- **Basis:** OpenTelemetry — Traces (a trace is a tree of spans, each span ≤1
  parent, many children, rooted at one root span); Session semantic conventions
  (a session is a collection of spans/logs/events over the session lifetime
  correlated by a session id); Trace semantic conventions (spans cover any
  instrumented operation — HTTP, DB, messaging, services, arbitrary ops).
- **Decision / Model:**
  1. **Hierarchy:** `Session/Conversation → one or more Traces → tree of spans`.
     A **turn = one trace**; a **session ⊇ many traces** (each user request is a
     trace; background actions are their own traces) — NOT 1:1. The PRD already
     supports this: `agent_executions.session_id` groups multiple executions/
     traces; `otlp_spans` carries both `session_id` and `trace_id`. (Corrects the
     earlier "turn = trace = session" shorthand.)
  2. **The span tree is NOT limited to agent nodes.** A trace contains spans for
     ANY instrumented operation — AGENT, MODEL (LLM), TOOL, RETRIEVAL, HTTP,
     DATABASE, messaging, FUNCTION — all sharing one `trace_id`, linked by
     `parent_span_id`, context-propagated across processes/services. (PRD
     `span_category` = AGENT/MODEL/TOOL/RETRIEVAL/ORCHESTRATION/OTHER; HTTP/DB map
     via OTel `span_kind`=CLIENT + OTHER, or as children of a TOOL span.)
  3. **Two derived views TAO should build — both valuable:**
     - **Execution Trace Tree** — the full OTel span tree (agent + model + tool +
       HTTP + DB + function). Powers the SRE waterfall (§5), latency breakdown
       (§6), first-failing-span. Answers *"what actually happened, where was the
       latency/failure."*
     - **Agent Execution Tree** — a **semantic projection** that collapses
       non-agent spans and shows only the agent hierarchy. Powers behavior /
       executive views. Answers *"which agents called which agents."*
  4. **Make explicit in the TAO data model + UI:**
     - `Session → one or more traces → tree of spans`.
     - The **agent hierarchy is a semantic projection of the span tree, not the
       span tree itself.**
- **Impact:**
  - AD-002's topology derivation IS the Agent-Execution-Tree projection: attribute
    each non-agent span (model/tool/retrieval/HTTP/DB) to its **nearest ancestor
    AGENT span**; agent→agent edges from nested agent spans. `agent_dependencies`
    (§11.2.9) = cross-trace aggregation of this projection.
  - **UI:** offer both — the full waterfall (Execution Trace Tree) and the
    collapsed Agent Tree — with toggle/drill between them.
  - **Data model:** keep `session_id` distinct from `trace_id` (1:many); do not
    assume one trace per session.
- **Status:** Accepted.

### AD-005 — Topology: storage and retrieval (per-trace tree vs aggregate graph)

- **Date:** 2026-08-12
- **Area / Topic:** Topology display, data model, sweepers vs on-read
- **Relationship:** CLARIFIES / ADDS — §4.3 (Dependencies tab), §5 (impacted
  systems), §11.2.9 (`agent_dependencies`), §11.3.1 (`otlp_spans`), §13.1
  (execution modes), §13.3 (Dependency Reconciler)
- **Question:** Display topology (as a waterfall with nodes?). Capture the entire
  tree in a new class with `rootAgentId`? Need another sweeper? Any way to fetch
  the topology tree other than sweeping?
- **Decision:**
  1. **Two distinct topologies, different storage:**
     - **Per-execution tree** (one trace's Execution/Agent tree) — IMPLICIT in
       `otlp_spans` (`trace_id` + `parent_span_id`). **No new class** —
       reconstruct on read. Always current, no drift.
     - **Aggregate topology** (agent ↔ models/tools/systems/sub-agents over time;
       the §4.3 graph) — materialized as **`agent_dependencies`** (edges, keyed by
       `agent_id`, `parent_dependency_id`). Already in the PRD — not a new class.
  2. **New class?** No for per-trace; the aggregate already exists as
     `agent_dependencies`. **Optional** render cache: materialize a per-execution
     collapsed **Agent Tree** as JSONB on `agent_executions` at ingest
     (`root_agent_id` attribute) — a convenience only; source of truth stays the
     spans. Add only if on-read projection proves too slow.
  3. **New sweeper?** No — the **Dependency Reconciler** (§13.3) already builds/
     updates `agent_dependencies`. Per-trace trees need no sweeper.
  4. **Fetching topology without sweeping:**
     - **On-read reconstruction** — per-trace tree via a **recursive CTE** over
       `otlp_spans` (or app-side parent/child assembly). Primary non-sweeping
       path; always live.
     - ~~Materialized view over spans for the aggregate graph~~ — **REMOVED (see
       AD-006):** Postgres is POC-only; production is a columnar store where this
       is not viable.
     - **Incremental upsert at ingest** — a near-real-time processor (§13.1
       mode 1) upserts `agent_dependencies` edges as each trace lands (bump
       `last_seen`), instead of periodic sweeping.
     - **REGISTERED source** — derive from the agent's manifest/definition when
       Wendy's agents declare tools/sub-agents (no telemetry needed); complements
       OBSERVED per the `discovery_source` enum.
  5. **Visualization:** waterfall = the **per-trace** view (time-ordered; latency/
     failure) = Execution Trace Tree for one trace. Aggregate topology (§4.3) =
     **node-link graph**, not a waterfall (relationships, not a timeline).
- **Recommendation:** per-trace = on-read by `trace_id` (no new class); aggregate
  = `agent_dependencies` via the Dependency Reconciler; REGISTERED as a
  complement. Optional per-execution agent-tree JSONB cache only if render perf
  demands it.
- **Status:** **Amended by AD-006** (production scale): materialized-view option
  removed; aggregate topology = sweeper-based incremental aggregation; per-trace
  on-read retained.

### AD-006 — Production scale & storage: sweeper-based incremental aggregation

- **Date:** 2026-08-12
- **Area / Topic:** Storage, scale, aggregation strategy
- **Relationship:** AMENDS **AD-005**; CLARIFIES §11.1 (Postgres is POC-only),
  §13.1 / §13.3 / §13.6 (execution modes, sweepers, incremental windows)
- **Context / constraints:**
  1. **Postgres is the POC/temporary store only.** Production storage is a
     **columnar DB**, accessed via **new Trillo AOS APIs** (to be added). Design
     must not depend on Postgres-specific features.
  2. **Scale: several million events/minute**; span/event tables grow very large.
     Aggregate queries over raw spans on every read are **not viable**.
  3. **Materialized views are OUT** (removes the AD-005 option) — not appropriate
     on the columnar production store.
- **Decision:**
  1. **Aggregate topology is built by SWEEPERS** (incremental aggregation) — not
     on-read queries, not DB materialized views. `agent_dependencies` is the
     accumulator, updated from swept recent records.
  2. **Incremental over recent records** — sweepers process only records added
     since a **watermark** (§13.6 job model); no historical rescans.
  3. **Union over time** — an agent's subnode/dependency set is **accumulated
     across traces**, since an agent may emit different tool/subnode sets in
     different traces (A `{X,Y}`, B `{X,Z}` → `{X,Y,Z}`). Sweepers **UPSERT**
     edges (agent→target) with `first_seen`/`last_seen`; topology is the growing
     union, never a per-trace or per-query snapshot.
  4. **Future: one combined sweep** — topology, rollups, findings, cost/token
     aggregations fan out from a **single pass over recently-added records** (read
     recent data once; pluggable aggregators). Design the sweeper contract for
     this now.
  5. **Per-trace tree unchanged** — viewing ONE execution's waterfall is a
     targeted lookup by `trace_id` (bounded span set) and stays **on-read**, no
     sweeper — provided the store is indexed/partitioned by `trace_id`.
  6. **REGISTERED source** (agent manifests) remains a valid complement (no
     telemetry), per AD-002 / AD-005.
- **Impact:** abstract the telemetry store behind Trillo AOS APIs (columnar in
  prod, Postgres in POC); the sweeper contract is watermark-incremental and hosts
  multiple aggregators in one pass; `agent_dependencies` upsert-accumulates the
  union with first/last-seen.
- **Status:** Accepted.

### AD-007 — Dependency retirement: last-seen, query-side, deferred

- **Date:** 2026-08-12
- **Area / Topic:** Dependency lifecycle / topology staleness
- **Relationship:** CLARIFIES — §11.2.9 (`agent_dependencies.last_seen_at`,
  `is_active`)
- **Decision:** Record `last_seen` on every dependency edge. Retirement is
  handled **at query time** — filter out edges not seen within a recency window —
  rather than a materialized state flip. **Go easy for now: not built in V1**;
  edges accumulate (the union), and the recency filter is added later when the
  topology query needs it.
- **Status:** Accepted (deferred build).

### AD-008 — Telemetry simulator requirements (companion document)

- **Date:** 2026-08-12
- **Area / Topic:** Synthetic data generation for the POC
- **Relationship:** ADDS — expands §11.5 (Synthetic Dataset Requirements)
- **Decision:** The POC's telemetry is produced by a **Telemetry Simulator** that
  stands in for real agent emission (in production, agents emit; for the POC, the
  simulator generates for N agents). Requirements captured in the companion
  document **`Enterprise_AI_Agent_Observability_POC_Telemetry_Simulator_Requirements.md`**.
  Conforms to the PRD data model + AD-001..AD-007.
- **Status:** Draft (see companion document).

### AD-009 — Status model + POC Application & UX Design (companion document)

- **Date:** 2026-08-12
- **Area / Topic:** Agent/instance/location status, inventory & dependency logic, UX
- **Relationship:** ADDS / CLARIFIES — §11.2.4 / §11.2.5 (`agents` / `agent_instances`
  `status`), §4–§5 (inventory, reliability), §11.2.9 (dependencies)
- **Decision:**
  1. **Status = latest trace status** (freshest signal), materialized on
     `agents.status` / `agent_instances.status` by the sweeper each run.
  2. **Trace status = highest severity** across its spans/events:
     `ERROR > WARNING > HEALTHY` (warning-but-no-error ⇒ `WARNING`).
  3. **Instance status** = last-trace status of that instance; **Location status**
     = **worst** status among instances at that location.
  4. **Error rate is a separate rollup** (trend), not the status badge.
  5. **Agent-level dependency view** = the aggregate `agent_dependencies`
     (union-over-time); **trace-level** = that trace's actual spans. Two different
     sources.
  6. Inventory-build (L1), dependency-build (L2), agents-by-location + location
     status (L3), dependency-tree fetch (L4), and the UI (location map → agents/
     instances → agent view → dependency span-view → trace) are specified in the
     companion doc **`Enterprise_AI_Agent_Observability_POC_Application_and_UX_Design.md`**.
  7. **Code-vs-deployment classifier:** a failure cluster's **spread** (distinct
     instances/locations/versions) drives the triage hint — wide spread ⇒ likely
     code; concentrated ⇒ likely deployment.
- **Resolved (2026-08-12):** **logical-agent status = count-by-status**, derived
  **dynamically** (a `GROUP BY` over the materialized `agent_instances.status`,
  e.g. `12 failed / 30 warned / 4,210 healthy`), clickable to the reds. Compact
  single badge = worst-of-instances ("any red ⇒ red"). Same pattern reused at
  location and application level.
- **Status:** Status model **Accepted**; logical-agent-status **Accepted**
  (count-by-status, dynamic).

### AD-010 — Simulator emits telemetry; sweepers derive inventory

- **Date:** 2026-08-12
- **Area / Topic:** Simulator vs platform division of labor
- **Relationship:** AMENDS the Simulator Requirements (§2, §11); CLARIFIES
  §11.2.3–§11.2.9 (inventory entities), §13.3 (Inventory/Dependency Reconcilers)
- **Decision:**
  1. The **simulator emits ONLY telemetry** (traces/spans, metrics, events, logs);
     it does **not** pre-populate inventory tables.
  2. **Sweepers DERIVE** `applications`, logical `agents`, `agent_instances`,
     `models`, `tools`, `external_systems`, vector stores, and `agent_dependencies`
     (OBSERVED) from the telemetry — the production-faithful discovery path.
  3. **Business metadata** not intrinsic to operations (`owner_team`,
     `cost_center`, `business_unit`, `business_purpose`, `governance_class`,
     `model_tier`, system `criticality`) is carried as **OTel resource attributes**
     so the sweeper derives it (or, alternatively, seeded via a REGISTERED source).
     Deliberately omitting some ⇒ metadata-incomplete agents (demo).
  4. **`model_pricing`** is the one **reference table** seeded as dummy data (not
     telemetry).
- **Status:** Accepted.

### AD-011 — Inventory build covers all entities; dimension tables vs. attributes

- **Date:** 2026-08-13
- **Area / Topic:** Inventory derivation, star-schema modeling
- **Relationship:** CLARIFIES / AMENDS — L1 in the App/UX doc; §11.2.3–§11.2.8
  (`applications`/`models`/`tools`/`external_systems`), §11.2.11, §11.3
- **Question:** L1 only described building `agents`/`agent_instances`. How are
  `applications`, `models`, `tools`, `external_systems` (and vector stores) built —
  or are some just attributes?
- **Decision:**
  1. **Both, by design.** Model/tool/system/application values are denormalized as
     flat **attributes** on `otlp_spans`/`agent_executions` (PRD §11.3) for fast
     filter/group-by analytics. The sweeper *also* maintains **dimension tables**
     (distinct values + metadata not in telemetry) for the inventory catalog,
     dependency/topology views, ownership, and pricing joins. **Attributes power
     analytics; dimension tables power inventory + hold metadata.**
  2. **L1 builds all inventory entities** (one pass, shares span traversal with
     L2), each by extracting distinct keys from telemetry attributes and UPSERTing
     with metadata from resource attrs / seeded pricing:
     `applications` (from `application_id` + resource attrs), `agents`,
     `agent_instances`, `models` (distinct provider/model/version from MODEL spans,
     join `model_pricing`), `tools` (from TOOL spans' `tool_name`),
     `external_systems` (from `dependent_system` + HTTP/DB targets).
  3. **Vector stores** are modeled as **`external_systems`** rows
     (`system_type=vector_store`) — no separate table (PRD §11.2.11 has none) —
     surfaced via RETRIEVAL spans + `dependency_type=VECTOR_STORE` edges.
  4. Distinct-sets are small/bounded → dimension-table UPSERTs are cheap even at
     scale.
- **Status:** Accepted.

### AD-012 — Competitive positioning (companion document)

- **Date:** 2026-08-13
- **Area / Topic:** Competitive analysis / GTM positioning
- **Relationship:** ADDS — positioning context (not a change to the PRD)
- **Note:** Assessed TAO's spec (PRD + AD-001..AD-011) against the Phoenix ×
  Galileo review frontier. Summary: **Strong** on multi-agent/A2A observability
  and enterprise-ops/fleet/governance (our wedge); **Partial** on alerting,
  security evals, drift, retention/sampling, instrumentation polish (several
  closable via existing AOS primitives — webhooks/scheduler/authoring);
  **Gap** on true inline guardrails (a different product category) and prompt/
  dataset experimentation + regression (a different, dev persona). Full scorecard
  in **`Enterprise_AI_Agent_Observability_Competitive_Positioning.md`**.
- **Status:** Informational (companion document).

### AD-013 — SRS v1.1 reconciliation (companion document)

- **Date:** 2026-08-13
- **Area / Topic:** Reconciling SRS v1.1 with the addendum
- **Relationship:** ADDS — cross-check of `Enterprise_AI_Agent_Observability_SRS_Revised.md`
- **Summary:** SRS v1.1 embodies most addendum decisions but **conflicts on five**
  needing an authority call: **(B1)** storage — SRS PostgreSQL+materialized-views
  vs AD-006 columnar-prod + MV-out; **(B2)** status — SRS single materialized
  ACTIVE/DEGRADED/INACTIVE vs AD-009 last-trace + count-by-status; **(B3)**
  discovery — SRS ingest-time cache/UPSERT vs AD-006 sweepers (rec: hybrid);
  **(B4)** SRS drops separate `models`/`tools`/`external_systems` tables (edges +
  attrs only) vs AD-011 dimension tables; **(B5)** SRS observed/registered
  metadata split vs AD-010 resource-attrs. **Gaps in SRS:** analytical tables
  (`metric_rollups`/`platform_findings`/`analysis_baselines`/`sweeper_runs`/
  `ai_analyses`), spread classifier, edge deployment. **SRS improvements to adopt:**
  Arrow/WAL ingestion, `gen_ai.*` semconv, governance detail + tamper-evidence
  realism, `store_id` as location, SLO/health-calc transparency. Full matrix +
  naming table in **`Enterprise_AI_Agent_Observability_SRS_Addendum_Reconciliation.md`**.
- **Status:** Open — B1–B5 pending decision.

<!--
Entry template (copy per decision):

### AD-00X — <short title>

- **Date:** YYYY-MM-DD
- **Area / Topic:** <e.g., Trace modeling, Cost calculation, Governance>
- **Relationship:** ADDS | CLARIFIES | AMENDS — PRD §<section number(s)>
- **Question:** <the question raised>
- **Decision:** <what was decided>
- **Rationale:** <why>
- **Impact:** <schemas / scenarios / UX affected; downstream work>
- **Status:** Accepted | Provisional | Superseded by AD-00Y
-->

---

## 4. Change History

| Addendum Version | Date | Summary |
| :--- | :--- | :--- |
| 0.1 | 2026-08-12 | Document created; ready for Q&A. |
| 0.2 | 2026-08-12 | Added AD-001 (agent identity vs. version in telemetry). |
| 0.3 | 2026-08-12 | AD-001(a) resolved for V1 (agent emits `agent_id`); added AD-002 (deriving agent topology from span data). |
| 0.4 | 2026-08-12 | Added AD-003 (scope: observed agents are Wendy's generic-framework agents, not Trillo AOS); corrected AD-001(b) and AD-002 accordingly; added scope note to §1. |
| 0.5 | 2026-08-12 | Added AD-004 (session→1..N traces→span tree; span tree not agent-limited; Execution Trace Tree vs Agent Execution Tree projection); refined AD-002. |
| 0.6 | 2026-08-12 | Added AD-005 (topology storage/retrieval: per-trace tree on-read from spans vs aggregate `agent_dependencies`; no new sweeper; non-sweeping fetch options). |
| 0.7 | 2026-08-12 | Added AD-006 (production scale: columnar store via AOS APIs, sweeper-based incremental aggregation, union-over-time UPSERT, future combined sweep); amended AD-005 (removed materialized-view option). |
| 0.8 | 2026-08-12 | Added AD-007 (dependency retirement: last-seen, query-side, deferred) and AD-008 (telemetry simulator requirements — companion document created). |
| 0.9 | 2026-08-12 | Added AD-009 (status = latest trace status; trace status = worst severity; instance/location/logical status; error rate separate; agent-level dependency from `agent_dependencies` vs trace spans); created POC Application & UX Design companion doc (L1–L4 logic, U5–U9 UX, spread classifier). |
| 0.10 | 2026-08-13 | Resolved AD-009 logical-agent status = count-by-status (dynamic); added AD-010 (simulator emits telemetry only; sweepers derive inventory; business metadata via resource attrs; `model_pricing` seeded reference). Updated simulator §2 and App/UX §3/§7. |
| 0.11 | 2026-08-13 | Added AD-011 (L1 builds all inventory entities: applications/models/tools/external_systems; dimension tables + denormalized attributes coexist; vector stores = external_systems subtype). Expanded App/UX L1. |
| 0.12 | 2026-08-13 | AD-001(b) Accepted (agent emits semver/build tag as agent_version). No Provisional decisions remain. |
| 0.13 | 2026-08-13 | Added AD-012 (competitive positioning vs Phoenix x Galileo frontier); created Competitive Positioning companion doc. |
| 0.14 | 2026-08-13 | Added AD-013 (SRS v1.1 reconciliation): 5 conflicts (B1 storage, B2 status, B3 discovery, B4 model/tool/system tables, B5 metadata source), SRS gaps + improvements; matrix in companion doc. |
