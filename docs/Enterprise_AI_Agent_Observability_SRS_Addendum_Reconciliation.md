# SRS v1.1 ↔ Requirements Addendum — Reconciliation

**Document Version:** 0.1 (draft)
**Reconciles:** `Enterprise_AI_Agent_Observability_SRS_Revised.md` (SRS **v1.1**)
against the **Requirements Addendum** (AD-001..AD-012) and its companions
(Simulator Requirements, Application & UX Design).

**Authority question to resolve first:** is SRS v1.1 the new source of truth (so
conflicting addendum decisions align to it), or a parallel draft to reconcile?
Recommendations below assume we keep each doc's *stronger* choice and unify.

---

## A. Aligned / confirmed (SRS embodies the decision)

- **Logical vs. instance vs. observed/registered** (AD-001, AD-003) — SRS §1, §2.2.
- **`agent_id` stable; version = `service.version`** (AD-001) — SRS §2.2.
  *Note:* SRS uses **`gen_ai.agent.id` / `gen_ai.agent.name`** (GenAI semconv) —
  better than the `trillo.agent.id` example in AD-001(a); adopt it.
- **Topology from spans → `agent_dependencies` edges**, `discovery_source`
  (OBSERVED/REGISTERED/INFERRED), **union via `UNIQUE(agent_id,type,key)` +
  `last_seen`** (AD-002, AD-006 union, AD-007) — SRS §2.2, Appendix A.4.
- **Wendy's generic agents on GKE emitting OTLP** (AD-003) — SRS §2.1.
- **Trace/span/session dims; waterfall + topology graph** (AD-004) — SRS §2.3,
  Scenarios 1–2.
- **Effective-dated `model_pricing`, `cost_source`, `pricing_record_id` on span**
  — SRS §4.3, Appendix A.5/A.9.
- **Query-side retirement, last_seen present, not built** (AD-007) — SRS §4.3.

## B. Conflicts to resolve — NEED A DECISION

| # | Topic | SRS v1.1 says | Addendum says | Recommendation |
| :- | :- | :- | :- | :- |
| B1 | **Storage & aggregation** | PostgreSQL **primary**; **materialized views** for rollups (§4.4) | **AD-006:** Postgres = POC-only; prod = **columnar via AOS APIs**; **materialized views OUT** | Keep **AD-006** for production; frame SRS Appendix A as the **POC/logical** schema; treat materialized views as a POC convenience, not the prod mechanism (prod = sweeper incremental). |
| B2 | **Status model** | single materialized `operational_status` = ACTIVE/DEGRADED/INACTIVE on `agent_inventory` **and** instances | **AD-009:** status = **latest-trace** (HEALTHY/WARNING/FAILED); **logical = count-by-status (dynamic)**, not a single materialized value | Adopt **AD-009**: instance = last-trace; logical = count-by-status. Keep **INACTIVE** as a separate **staleness** axis (from `last_seen`), map **DEGRADED ↔ has-errors**. Unify vocabulary. |
| B3 | **Discovery timing** | inventory/dependency discovery **at ingest** (in-memory cache + UPSERT on miss) | **AD-006:** periodic **watermark sweepers** | **Hybrid (not exclusive):** ingest-time cache+UPSERT for low-cardinality **inventory/dependency** (SRS — the cache absorbs it at scale); **sweepers** for heavy **rollups/findings/baselines** (AD-006). |
| B4 | **Model/Tool/System tables** | **NO** separate `models`/`tools`/`external_systems` tables — represented as `agent_dependencies` edges + span attributes + `model_pricing` | **AD-011:** separate `models`/`tools`/`external_systems` **dimension tables** | **Decision.** SRS's leaner model works if no first-class cross-agent **catalog** is needed (pricing via `model_pricing`; unapproved-model via governance). Restore dimension tables **only** if a Models/Tools/Systems catalog view with typed metadata (tier, approval, criticality, owner) is required. **Lean: accept SRS simplification; amend AD-011.** |
| B5 | **Business-metadata source** | **observed** (telemetry) vs **registered** (CMDB/config/admin) split; business metadata from the **registered** source (§1, §2.2.8) | **AD-010:** business metadata carried as **OTel resource attributes** (telemetry) | Adopt **SRS's observed/registered split** (cleaner — don't cram owner/cost_center into every span). For the POC, **seed a registered-metadata fixture** instead of resource attrs. **Amend AD-010 + Simulator doc.** |

## C. Gaps — addendum/PRD items the SRS omits

- **Analytical / background tables** — `metric_rollups`, `platform_findings`,
  `analysis_baselines`, `sweeper_runs`, `ai_analyses` (PRD §11.4). Needed for the
  AD-006 sweeper contract, AD-009 findings, and the executive rollups. SRS §4.4
  mentions rollups generically but Appendix A has no schema for them. **Add.**
- **Spread / code-vs-deployment classifier** (AD-009, App/UX §4.5). SRS Scenario 2
  groups errors "by root cause, agent version, store, tool, system" (close!) but
  doesn't name **spread/blast-radius** or the classifier. **Add.**
- **Agent Execution Tree projection** naming (AD-004) — SRS has waterfall +
  topology graph but not the formal two-view framing. Minor.
- **Edge deployment** (AD-003 / Simulator) — SRS §2.1 shows GKE only. Minor —
  add edge as a deployment target.
- **HTTP / DATABASE `span_category`** (AD-004 open item) — SRS enum is still
  AGENT/MODEL/TOOL/RETRIEVAL/ORCHESTRATION/OTHER; downstream HTTP/DB unlabeled.
  Still open.

## D. Additive — SRS improvements to fold back into our set

- **Ingestion architecture** (§2.1): OTLP gRPC/HTTP → validate/normalize → **Arrow
  RecordBatches** → **durable WAL/write queue** → store. Adopt as the ingestion
  design (reconcile the store target with AD-006 columnar).
- **`gen_ai.*` semconv naming** — adopt over `trillo.agent.id` (update AD-001a).
- **Governance detail** — `governance_policy_versions` table, `governance_decisions`,
  `content_hash`, **tamper-evidence realism** (§6.4: "shall not describe mutable
  Postgres rows as immutable"), admin-audit **hash chaining**. Adopt.
- **`store_id`** as the concrete **location** dimension (Wendy's 7,420 stores) —
  reconcile with our `location_id`/region (region = a grouping of stores).
- **Transparency**: health-status calculation exposure (§7.3), **SLO honesty**
  (§4.4: define SLOs after benchmarking), latency breakdown incl. **queueing /
  framework overhead** (§3.1 — supports the two-span client/server model).
- **Data-quality surface** (§4.3): instrumentation coverage, estimated-cost
  marking, observed/registered/inferred dependency marking. Adopt.

## E. Naming to unify across all docs

| Concept | PRD / Addendum | SRS v1.1 | Pick |
| :- | :- | :- | :- |
| Logical agent table | `agents` | `agent_inventory` | (decide) |
| Governance outcome | `governance_evaluations` | `governance_decisions` | `governance_decisions` (clearer) |
| Admin audit | `administrative_audit` | `administrative_audit_events` | (align) |
| Location | `location_id` / region | `store_id` (+ region grouping) | `store_id` + region rollup |
| Agent identity attr | `trillo.agent.id` | `gen_ai.agent.id` | `gen_ai.agent.id` |

## F. Net decisions needed (from §B) + follow-ups

1. **B1** storage authority (columnar-prod vs Postgres+MV).
2. **B2** status model (count-by-status/last-trace vs materialized ACTIVE/DEGRADED/INACTIVE).
3. **B3** discovery timing (hybrid ingest+sweeper — recommended).
4. **B4** keep or drop separate models/tools/systems tables.
5. **B5** metadata source (observed/registered split vs resource-attrs).
6. Fold in §C gaps (analytical tables, spread classifier), §D additive, §E naming.
