# OliverDB ⇄ OTel Mapping & Agent-Identity Resolution — Requirements

**Document Version:** 0.1 (draft)
**Status:** For discussion with the **OliverDB team** — candidate to run **in-process**
as an OliverDB **Rust plugin** (OliverDB supports plug-ins), vs. an app-side sweeper.
**Companion to:** PRD v1.5, SRS v1.1, Requirements Addendum (AD-001, AD-010),
Gap Analysis & OliverDB Interface (§3.D).

Covers backlog items **G1** (agent-identity resolve-by-name fallback) and **G5**
(framework conformance / OTLP→`gen_ai.*` mapping + instrumentation-coverage report).

---

## 1. Purpose

The observed agents are **generic-framework** agents (LangChain, CrewAI,
LlamaIndex, raw Vertex/Bedrock/OpenAI SDKs — Addendum AD-003). They emit
OpenTelemetry, but **what they emit varies by framework/instrumentation library**
(OpenLLMetry vs OpenInference vs SDK-native), and some do **not** emit the fields
TAO's analytics depend on. Two consequences we must engineer for:

1. **Identity (G1):** TAO V1 assumes agents emit a stable `gen_ai.agent.id`
   (AD-001a). Many won't. We need a **resolve-by-name fallback** so inventory
   still works.
2. **Conformance (G5):** TAO's sweepers expect a canonical `gen_ai.*` shape. We
   need a **normalization/mapping layer** that maps each framework's dialect into
   that shape, and a **coverage report** that tells the customer exactly which
   required fields their instrumentation is missing.

**We do not control how the customer builds their agents.** So this is *not* a
"ship our own tracing library" effort — the value is the **ingestion-side mapping
+ conformance/coverage**, which is ours regardless of the customer's emitter.
(This supersedes the earlier "one-line auto-instrumentation libraries" framing of
G5; a thin optional shim to add missing fields is a secondary, nice-to-have.)

## 2. Where it should run (the key discussion with OliverDB)

The mapping + identity resolution is **per-record, high-volume, deterministic** —
exactly the profile that wants to run **at ingestion, in-process**:

- **Option A — OliverDB in-process Rust plugin (preferred to evaluate):** OliverDB
  ingests OTLP → Arrow; a plugin normalizes attributes and resolves identity
  **before** the RecordBatch is stored, so everything downstream sees the
  canonical shape. Lowest latency, no second pass, no re-write of stored rows.
- **Option B — app-side sweeper (fallback):** a Trillo AOS sweeper post-processes
  stored rows into canonical columns. Works, but adds a pass and storage churn;
  only if the plugin path isn't viable.

**Question for OliverDB:** can a plugin (Rust) run a per-record transform in the
ingest path — read resource/span attributes, write normalized columns, and do a
cheap cache-backed lookup (for identity resolution) — within the ingest latency
budget? What's the plugin API surface (per-record vs per-batch, state/cache,
external lookup)?

## 3. G1 — Agent-identity resolution

### 3.1 Requirement
Resolve every telemetry record to a stable **logical `agent_id`** even when the
agent does not emit `gen_ai.agent.id`.

Resolution order (first match wins):
1. **Emitted** `gen_ai.agent.id` (trusted; AD-001a V1 path).
2. **Resolve-by-name:** map `(gen_ai.agent.name | service.name) + application + environment`
   → a stable `agent_id`, via a **registry** the platform maintains
   (deterministic hash or an assigned id, stable across restarts).
3. **Unresolved:** stamp a synthesized id + flag the record/agent
   `identity_source = UNRESOLVED` so it surfaces as a metadata-quality finding
   (do not drop).

### 3.2 Notes
- The registry lookup must be **cache-backed** (hot set is small — tens of
  logical agents) so it doesn't cost a round-trip per span.
- Emit an `identity_source` marker (`EMITTED | RESOLVED | UNRESOLVED`) on the
  record so the app can show how identity was derived (data-quality, SRS §4.3).
- `agent_version` is likewise taken from `service.version` (AD-001b); if absent,
  mark `UNKNOWN` (still comparable, just labeled).

## 4. G5 — OTLP→`gen_ai.*` conformance & mapping

### 4.1 Required canonical fields (what TAO depends on)
The mapping layer must populate these; missing ones degrade specific features:

| Canonical field | Feeds | If missing |
| :-- | :-- | :-- |
| `agent_id` (+ `identity_source`) | Inventory, all pivots | see G1 |
| `agent_version` | Version pivots, A/B (Addendum2 AD-017), regression | comparisons labeled UNKNOWN |
| `session_id` | Session grouping, multi-turn, drift | session views degrade |
| `input_tokens` / `output_tokens` (/ cached / reasoning) | **Cost & token scenarios** | **cost silently breaks** — #1 risk |
| `request_model` / `response_model` / `provider` | Cost, model pivots | model analytics degrade |
| `span_category` (AGENT/MODEL/TOOL/RETRIEVAL/…) | Trace tree, latency breakdown, topology | breakdown/topology degrade |
| `tool_name` + `dependent_system` | Topology, impacted-systems | dependency graph incomplete |
| `trace_id`/`span_id`/`parent_span_id` | Everything | core — must be present |

### 4.2 Per-framework mapping
Maintain a **mapping table per source dialect** → canonical `gen_ai.*`:

- **OpenLLMetry** (common for LangChain) — `gen_ai.*` semconv, mostly aligned.
- **OpenInference** (Arize/LlamaIndex) — different attribute names; map to canonical.
- **SDK-native** (Vertex / Bedrock / OpenAI Agents SDK) — provider-specific; map
  token/usage + model attributes.
- **Unknown/custom** — pass through to `raw_attributes`, mark low coverage.

The mapping is **data-driven** (a config table), so a new dialect is added without
a code change — important because we can't predict every customer emitter.

### 4.3 Instrumentation-coverage report (the real adoption asset)
Produce, per **application/agent/instance**, a **coverage scorecard**: which
canonical fields are present vs missing, with the **business impact** of each gap
spelled out ("LangChain agents emit everything except `*_tokens` → **cost
analytics unavailable** until token usage is emitted; add `<X>`").
- Drives the Inventory/Metadata **data-quality** surface (SRS §4.3).
- Turns "your telemetry is incomplete" into an actionable, specific fix list —
  the honest, defensible version of "onboarding help."

## 5. Data model touchpoints
- Canonical normalized columns on the telemetry rows (per §4.1) — ideally written
  at ingest (Option A) so no separate table.
- An **agent-identity registry** (name+app+env → agent_id) — small, in Cloud SQL
  metadata store; cached in the ingest plugin.
- A **dialect mapping config** (source attr → canonical) — data-driven.
- A **coverage scorecard** per agent/instance (derived; feeds data-quality UI).

## 6. Open questions for the OliverDB team
1. **Plugin execution model:** per-record vs per-RecordBatch transform in the
   ingest path? State/cache support for the identity registry? External lookup
   allowed, or must the registry be pushed into the plugin?
2. **Latency budget** for an in-process transform at target volume (millions/min).
3. **Schema authority:** does the plugin write **canonical columns** into the
   stored Arrow schema (so the app queries canonical directly), or only annotate?
   (Ties to Gap-Analysis §3.D "who owns OTLP→schema mapping.")
4. **Config hot-reload:** can the dialect-mapping config + identity registry be
   updated without a plugin redeploy?
5. **Fallback:** if the plugin path isn't viable, confirm the app-side sweeper
   (Option B) can read raw attributes + write canonical columns efficiently.

## 7. Relationship to the backlog
- **G1** = §3 (identity resolution). **G5** = §4 (mapping + coverage); the
  "ship framework emitters" idea is **dropped** in favor of ingestion-side mapping
  + coverage report (optional thin shim only if a customer needs to add a missing
  field).
- Directly reduces the **#1 POC-success risk** (instrumentation/token gaps,
  Gap-Analysis §5) and pulls the AD-001a **resolve-by-name** fallback forward.
