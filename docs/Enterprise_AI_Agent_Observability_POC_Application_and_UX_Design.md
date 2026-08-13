# Enterprise AI Agent Observability & Analytics
## POC Application & UX Design

**Document Version:** 0.3 (draft)
**Companion to:** POC Requirements v1.5 + Requirements Addendum (decision log)
**Scope:** The observability application (built on Trillo AOS) — its aggregation
logic and its user experience. Telemetry source = generic-framework agents
(Addendum AD-003); generated for the POC by the Telemetry Simulator.

---

## 1. Purpose

Specifies (A) the platform **logic** that turns raw telemetry into inventory,
status, dependencies, and location rollups, and (B) the **UX** for navigating from
locations → agents/instances → agent view → dependency/trace evidence.

## 2. Core model: Agent, Instance, Location

- **Logical Agent** — the code/definition (prompt, tools, model, logic); identical
  across deployments. Inventory operates primarily on this (§11.2.4).
- **Agent Instance** — one runtime deployment of a logical agent at one location
  (GKE sandbox pod / edge). Executions and traces happen here (§11.2.5).
- **Location** — a region/site (East/West/South/North/Central, extensible). A
  location hosts **instances** (of possibly several logical agents).

**Triage axis (why the split matters):** a failure isolated to one
instance/location/cluster ⇒ **deployment/environment** issue; the same failure
signature spread across many instances of one logical agent ⇒ **code/logic/model**
issue (usually version-correlated). The app surfaces this via a failure cluster's
**spread** (see §4.5).

## 3. Status model (AD-009)

- **Span severity:** `ERROR` (span error / exception), else `WARNING` (a
  correlated warning-level signal — non-blocking eval FAIL, policy `WARN`, or
  WARN log — with no error), else `HEALTHY/OK`.
- **Trace status** = the **highest severity** across the trace's spans/events:
  `ERROR > WARNING > HEALTHY`. (A trace with a warning but no error = `WARNING`.)
- **Instance status** = the status of that instance's **latest trace** (by end
  time). It is the **freshest** signal, materialized on `agent_instances.status`
  and refreshed by the sweeper each run.
- **Location status** = the **worst** (highest severity) status among the
  instances at that location.
- **Error rate** is a **separate rollup** (Reliability Health Sweeper) — a trend
  metric, NOT the status badge.
- **Logical-agent status** — **not a single materialized value**; derived
  **dynamically** as a **count-by-status** over its instances (a `GROUP BY` over
  `agent_instances.status`, e.g. `12 failed / 30 warned / 4,210 healthy`),
  clickable to drill to the instances of a given status. A compact single badge,
  when needed, uses **worst-of-instances** ("any red ⇒ red"). The same
  count-by-status pattern applies at **location** and **application** level. Cheap
  — it aggregates the materialized instance statuses, not raw spans.

## 4. Platform logic

### 4.1 (L1) Inventory build — sweeper

Incremental sweep of telemetry since the watermark (Addendum AD-006). Every
inventory entity is built by extracting **distinct keys from telemetry
attributes** and **UPSERT**ing, enriched with metadata carried on **resource
attributes** (AD-010) or the seeded `model_pricing` reference. All UPSERTs are
incremental (watermark) and cheap — the distinct-sets are small and bounded.

**Dimension tables vs. attributes (design note).** Model/tool/system/application
values are ALSO denormalized as flat attributes on `otlp_spans` /
`agent_executions` (PRD §11.3) to drive fast filter/group-by analytics. The
sweeper *additionally* maintains **dimension tables** — distinct values + the
metadata that telemetry doesn't carry — for the inventory catalog, dependency/
topology views, ownership, and pricing joins. **Attributes power analytics;
dimension tables power inventory + hold metadata.** Both coexist by design.

Entities built (one pass; shares the span traversal with L2):

- **`applications`** — distinct `application_id` (+ `application_name`); metadata
  (`business_unit`, `owner_team`, `cost_center`, `environment`) from **resource
  attributes**. UPSERT; app-level status = count-by-status over its agents
  (dynamic, §3).
- **`agents`** (logical) — distinct `(agent_id, agent_name, application_id)` →
  UPSERT; set `current_version` (latest `agent_version` seen), `last_seen_at`,
  `status` (§3); `owner_team`/`business_purpose`/`cost_center`/`governance_class`
  from resource attrs; **metadata-completeness flags** when those are absent (the
  Inventory/Metadata scenario).
- **`agent_instances`** — distinct `(agent_id, service_instance_id, location_id,
  gcp_project_id/cluster/namespace/pod OR edge id, agent_version, environment)` →
  UPSERT; `first_seen_at`/`last_seen_at`; `status` = last-trace status of that
  instance (§3). One logical agent → many instances.
- **`models`** — distinct `(provider_name, model_name, model_version)` from
  `MODEL` spans (`request_model`/`response_model`); **join `model_pricing`**
  (seeded) for cost; `model_tier`/`approved_for_use` from resource attrs/seed.
  UPSERT.
- **`tools`** — distinct `tool_name` from `TOOL` spans; `tool_type`/`owner_team`
  from attrs; `external_system_id` from the observed tool→system pairing
  (`dependent_system`). UPSERT.
- **`external_systems`** — distinct `dependent_system` (from `TOOL` spans) plus
  downstream HTTP/DB targets; `system_type`/`criticality`/`owner_team` from
  attrs/seed. UPSERT.
- **Vector stores** — surfaced via `RETRIEVAL` spans + `dependency_type=
  VECTOR_STORE` edges; modeled as **`external_systems`** rows
  (`system_type=vector_store`), not a separate table (PRD §11.2.11 has none).

Identity/existence is **derived from telemetry**; business metadata rides in
**resource attributes** (or the seeded pricing reference); omissions become
metadata-completeness findings.

### 4.2 (L2) Agent-dependencies build — sweeper

Per Addendum AD-002 / AD-005 / AD-006: project each trace's span tree onto agents
(attribute each non-agent span — model/tool/retrieval/HTTP/DB — to its **nearest
ancestor AGENT span**; agent→agent edges from nested agent spans), then **UPSERT**
edges into `agent_dependencies` accumulating the **union over time** with
`first_seen_at`/`last_seen_at`. Tool spans contribute the chained **tool→system**
edge via `dependent_system`. Retirement is query-side on `last_seen` (AD-007).

### 4.3 (L3) Agents grouped by location + location status

From `agent_instances` (each has `location_id`, `status`, `agent_id`):
- Group by `location_id`. For each location output:
  `{ location_id, distinct_agent_count, instance_count, location_status }` where
  `location_status = MAX_severity(status of instances at that location)`
  (`ERROR > WARNING > HEALTHY`).
- Drill targets: the **instances** at that location, and the **distinct logical
  agents** present there.
- Powers the location map (§5.1). (Materialized by the sweeper for scale, or
  fetched via the AOS API over the columnar store — never a raw-span scan.)

### 4.4 (L4) Agent dependency tree — fetch

From `agent_dependencies` for the selected `agent_id`: return its edges (models,
tools, systems, vector stores, sub-agents), **recursing through agent→agent
edges** to build the subtree (cycle-guarded), filtered to **active** edges
(query-side `last_seen` recency, AD-007). This is the **aggregate** topology
("what this agent connects to over time"), rendered as a span-style tree (§5.4).
Distinct from a **trace's** span tree (one execution, from `otlp_spans`).

### 4.5 (Support) Failure cluster + spread classifier

Failure Clusterer (§13.3) groups failures by signature (error type / first-failing
span / dependency). Each cluster carries **spread**: distinct **instances**,
**locations**, **versions** affected. High cross-instance/cross-location spread ⇒
flag **likely code/logic** issue; concentration in one instance/location ⇒ likely
**deployment/environment** issue. This drives the triage hint in the agent/SRE
views.

## 5. User experience

### Navigation model

```
Location Map ──► Agents/Instances at Location ──► Agent View ──► Dependency View (aggregate, span-style)
                                                        │
                                                        └──► Instance View ──► Trace View (span waterfall)
```
Breadcrumb is always visible: `Location ▸ Agent ▸ Instance ▸ Trace`.

### 5.1 (U5) Location map → agents at a location

- Draw agents **by location**; each location node **colored by `location_status`**
  (worst-of-instances, §4.3).
- **Click a location** → the **agent list filtered to that location** (distinct
  logical agents present there; each row shows how many instances at this
  location + that agent's status here). From a row, drill to the agent, or to the
  instances at this location.

### 5.2 (U6) Agent view (logical)

Selecting an agent opens the **Agent View**: header (owner, contact, purpose,
application, cost center, `current_version`, #instances, #locations, aggregate
status); tabs for **Dependencies** (§5.4), **Instances** (§11.2.5), **Reliability/
Latency/Cost** aggregates, and **Failure Clusters** with spread (§4.5). Trace lists
are **not** shown at this level (see §5.5).

### 5.3 (U9) Distinguishing Agent List vs Agent Instance View

Because the two look similar, differentiate explicitly:
- **Color-coded top bar** = status (red/amber/green) — instance = last-trace,
  agent = §3 badge.
- **Icon + title per level:** Agent = capability glyph + `OrderProcessingAgent`;
  Instance = deployment/pod glyph + `OrderProcessingAgent @ West · cluster-3 · v1.4`.
- **Level badge** ("LOGICAL" vs "INSTANCE") + breadcrumb.
- **Different dimensions:** Agent → owner, purpose, current_version, #instances,
  #locations, aggregate status. Instance → location, cluster/pod, version,
  environment, this-instance status, last-seen.

### 5.4 (U7) Agent dependency view — span-style (aggregate)

Render the L4 aggregate dependency tree as an **indented, span-style** visual
(agent → model / tool → system / sub-agent → …). This answers *"what this agent
connects to."* It is sourced from `agent_dependencies`, **not** a trace. The
**trace-level** span view (§5.6) is the separate "what happened in one execution."

### 5.5 (U10/U11) Trace scoping — agent vs instance

- **Agent (logical) level:** aggregates + curated entry points (failure clusters
  with spread, worst instances/locations, a couple of exemplar traces). **No raw
  per-agent trace list** (thousands of instances = firehose).
- **Instance level:** that instance's recent executions/traces (bounded).
- Troubleshooting a failed agent flows: agent → failure cluster (or worst
  instance/location) → **trace view**, always entered with context. Cross-instance
  spread routes toward a **code** hypothesis; single-instance toward
  **deployment** (§4.5).

### 5.6 Trace view

One execution's **span waterfall** (Execution Trace Tree, Addendum AD-004): time-
ordered spans with duration/status; first-failing span highlighted; drill into
prompt/model/tool args/response/exception/logs (§5.3 of PRD).

## 6. Data touchpoints

`agents`, `agent_instances`, `agent_dependencies`, `otlp_spans` (per-trace),
`platform_findings` / failure clusters, `metric_rollups` (error rate, latency,
cost). All reads go through the Trillo AOS APIs over the store (columnar in prod,
Postgres in POC).

## 7. Open items

- Warning-severity sourcing (which events/logs/policies map to `WARNING`).
- Location taxonomy beyond East/West/South/North/Central.
- Whether the location map is geographic vs. a grouped/region layout.
