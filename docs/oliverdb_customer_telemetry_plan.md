# OliverDB — Customer Telemetry Alignment Plan

**Document Version:** 0.3
**Purpose:** capture the small set of schema + doc changes needed to make the OliverDB side of the telemetry stack accept telemetry from customer agents built on ADK / LangChain / LangGraph / CrewAI (and any other framework a customer might use) while preserving Trillo Observability's `executionId`-based query model.
**Companion to:**
- `oliverdb_entities/OtlpSpan.json` / `OtlpLog.json` / `OtlpEvent.json` / `OtlpMetric.json` / `OtlpTelemetry.json` — target schemas for OliverDB
- `oliverdb_ingest_normalization_note.md` — hand-to-OliverDB brief (see §9 for the derivation rules)
- `ingest_to_oliverdb_from_agent_frameworks.md` — customer-facing framework recipes
- `app_team_oliverdb_guide.md` — Trillo-app developer guide

## 1. The core decision

**`executionId` stays as a first-class column on all OliverDB telemetry schemas.**

> **The purpose of `executionId` is to avoid `traceId` collisions across services / producers within a tenant, and to handle cases where the `traceId` emitted by a customer agent is non-OTel-compliant (short id, hard-coded value, framework wrapper bug).**

Two rationales, both defensible:

1. **Collision protection.** At 20K agents across mixed frameworks, we can't rely on every customer producer generating OTel-spec 128-bit random `traceId`. Some frameworks use shorter ids; some producers ship with hard-coded/test values; some have propagation bugs. `executionId` is a deterministically-derived, always-unique-within-tenant identifier that survives all of those failure modes.
2. **Trillo compatibility.** Trillo Observability's reader dispatchers, function code, and dashboards already query on `executionId`. Keeping it as a first-class column means no downstream refactor — the collision-protection story is what we tell OTel purists; the platform-compat story is what makes this an easy call for us.

Path decided:
- No changes to Trillo AOS platform code (`trillo-aos`, `tcs-service`, `tcs-ai`, `aos-py-execution`, `tcs-metadata aos_toolkit`).
- Two derivation rules in the OliverDB ingest plugin (spelled out in `oliverdb_ingest_normalization_note.md` §9):
  - Rule A: `traceId` ← `attrs.trillo.execution_id` when `traceId` is null (Trillo-scoped, no-op on customer telemetry).
  - Rule B: `executionId` ← promote `attrs.trillo.execution_id` if set, else derive as `UUIDv5(trillo_ns, service.namespace || traceId)` when `traceId` is set, else null.

## 2. Changes needed

### 2.1  OliverDB schemas (this repo's companion — `TrilloAgentObservability/.trillo/568/oliverdb_entities/`)

Restore `executionId` on the schemas that carry trace context. Add a leading bold note in each schema's top description stating the purpose.

| File | Change |
|---|---|
| `OtlpSpan.json` | `executionId` restored + description explains derivation; top-of-file description carries the bold "purpose" note |
| `OtlpLog.json` | Same |
| `OtlpEvent.json` | Same |
| `OtlpTelemetry.json` (unified) | Same |
| `OtlpMetric.json` | `executionId` **added** (didn't have one before) — populated only when `attrs.trillo.execution_id` is set on the producer, else null. Generic OTel metrics don't carry trace context at the data-point level; trace correlation lives inside per-exemplar records. |

### 2.2  OliverDB team note (this repo — `trillo-observability/docs/oliverdb_ingest_normalization_note.md`)

Revised §9 covers both rules end-to-end:
- 9.1 — collision picture (birthday math + non-compliance risks)
- 9.2 — the two derivation rules (with UUIDv5 justification and namespace UUID)
- 9.3 — ordering on the ingest path
- 9.4 — the questions we need the OliverDB team to answer (UUIDv5 feasibility, rule composability)

Bumped to v0.3.

### 2.3  Trillo AOS platform code

**No changes.** Every Java + Python path keeps its existing `executionId` naming and code flow. Trillo-emitted rows land in OliverDB with `attrs.trillo.execution_id` set; the plugin promotes it into the `executionId` column. Query consumers on the Trillo side continue reading `executionId`.

### 2.4  App-team living-reference guide — `app_team_oliverdb_guide.md`

Update the "Trillo execution correlation" row in the producer-discipline table (§3.3) to reflect the new derivation: OliverDB now has `executionId` as a first-class column; the plugin populates it (promote for Trillo, UUIDv5-derive for customer telemetry); consumers filter by `executionId` regardless of origin.

## 3. TODO

### 3.1 — Trillo platform team

- [x] **T-1** — Restore `executionId` column on the 4 OliverDB schemas (with new derivation-focused description). Add executionId to `OtlpMetric.json`.
- [x] **T-2** — Update `oliverdb_ingest_normalization_note.md` §9 with the two rules + collision picture. Bump to v0.3.
- [x] **T-3** — Update `app_team_oliverdb_guide.md` producer-discipline row.
- [x] **T-4** — Update this plan doc (v0.3).
- [ ] **T-5** — Mint the fixed `trillo_execution_id_namespace` UUID (a single value we hard-code in the plugin config). Include it when we share the note with the OliverDB team.
- [ ] **T-6** — Share the updated `oliverdb_ingest_normalization_note.md` with the OliverDB team when we next sync.

### 3.2 — OliverDB team (waiting on)

- [ ] **O-1** — Confirm UUIDv5 (RFC 4122, SHA-1-based) is feasible in the plugin path (Rust `uuid` crate supports it); or, if a raw hash is preferred, confirm the shape.
- [ ] **O-2** — Confirm the two rules (attribute promotion + derivation from two columns) compose in the plugin surface — declarative rules or a small Rust plugin, whichever is cleaner.
- [ ] **O-3** — Confirm ingest-plugin timeline (per the §8 questions in the note).

### 3.3 — Deferred (not required for this pass)

- Reader-dispatcher optimization: the four Trillo reader functions currently query via `attrs.trillo.execution_id`. Once the plugin populates the typed `executionId` column, they can switch to `WHERE executionId = X` (typed column, faster). Optional perf polish; existing queries continue to work.
- `sessionId`, `userId`, `genaiOperationName` as additional flat columns. Punt until a query needs them.

## 4. Estimated total effort

- Schema edits: done.
- Note update: done.
- Plan-doc update: done.
- App-team guide update: done.
- Mint namespace UUID + share note with OliverDB team: ~5 minutes.

**Wall-clock total for this pass:** everything except T-5 / T-6 is done.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-09-03 | Trillo (via Claude Code session) | Initial plan with `executionId` derivation ladder + emitter + reader-dispatcher rewrites. Superseded. |
| 0.2 | 2026-09-03 | Trillo (via Claude Code session) | Narrowed scope to drop `executionId` column and rely on `traceId`. Superseded. |
| 0.3 | 2026-09-03 | Trillo (via Claude Code session) | Restored `executionId` as first-class with collision-protection rationale + UUIDv5(namespace, traceId) derivation. Two plugin rules; no Trillo AOS platform-code changes. Added `executionId` to `OtlpMetric.json` (was missing). |
