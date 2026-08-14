# Gap Analysis, OliverDB Interface Questions & Proposal Risk Notes

**Document Version:** 0.1 (working / draft)
**Status:** Open — **revisit after the OliverDB team discussion.** Several proposal
claims (see §6) are *contingent* on partner answers; this doc flags what to
confirm and what to adjust/remove if a capability isn't available.
**Companion to:** PRD v1.5, SRS v1.1, Requirements Addendum, the OliverDB-powered
evaluation proposal.

---

## 1. How to use this document

Assume everything specified is implemented **except** the three pre-demo features
in AD-014 (Failure Spread, Security Evaluations, Alerting). This doc lists the
remaining gaps, splits them into **partner-confirm (OliverDB)** vs **platform
(ours)**, and maps the **proposal claims that hinge on partner answers** so we can
edit the proposal deliberately after the OliverDB discussion.

## 2. Architecture boundary (clarified by the proposal)

Two stores, clean seam:
- **OliverDB** — raw telemetry (spans/metrics/events/logs), Apache **Arrow**-based
  (columnar; OTLP via Arrow RecordBatches; query almost certainly **Arrow Flight
  SQL**). Partner owns ingest + redaction + RBAC tagging + storage + query.
- **Cloud SQL / PostgreSQL** — derived metadata: inventory, dependencies, pricing,
  governance policy/decisions, audit, findings, alerts. **This is where the AOS
  classes live** (singular, `*_tbl`, per AD-015).

Everything **up to and including "store for query" is OliverDB**; everything
downstream (sweepers, dashboards, governance app) is ours. This split also
**resolves reconciliation B1** (it's a hybrid, not columnar-vs-Postgres).

## 3. OliverDB interface questionnaire (confirm with partner)

### A. Query interface & compute
- Is the query interface **Arrow Flight SQL**? What SQL **dialect/completeness**
  (GROUP BY, **percentiles / approx-distinct**, window functions, joins)?
- Does OliverDB do **server-side aggregation / predicate push-down**, or only
  stream raw RecordBatches? *(At millions/min the ~25 sweepers must push
  aggregation down; pulling raw Arrow to GKE to aggregate would collapse.)*
- **Interactive latency** target for dashboards (sub-second) and **concurrency**
  limits (dashboards + ~25 sweepers in parallel).
- **Indexing/partitioning** for the two access patterns: point-lookup by
  `trace_id` (waterfall) AND time-range + `agent_id`/`store_id` scans (rollups).
  Arrow gives columnar scans but neither pattern is automatic. ("Ingest and index
  once" — proposal §3.3 — confirm what indexes exist.)

### B. Ingestion cursor & completeness (the key design item)
- **Preferred:** a **monotonic `long` seq per row**. If so:
  - Is it monotonic in **commit/visibility order** (safe) or **allocation order**
    (a low seq can become visible after a higher one → a `seq > last` scan skips
    it)?
  - **Global or per-partition**? (Per-partition ⇒ one cursor per partition.)
  - Can you also expose a **committed high-watermark `W`** (all `seq ≤ W`
    guaranteed visible)?
- If no seq: a **snapshot/version cursor** (Iceberg/Delta/**Lance**-style)?
- If neither: a **server-assigned ingest timestamp** — confirm it's stamped at
  **commit (not event time)**, its **resolution**, and the **max commit/visibility
  lag δ** (so we can size a trailing safety lag).

### C. Redaction & RBAC masking ⚠️ (highest-severity)
- **Redaction reversibility:** is ingestion redaction **destructive**, or
  **reversible** (tokenization / envelope-encryption) so authorized roles can
  **unmask**? *(The spec/proposal promise field-level unmask with permission — if
  redaction is destructive, that promise cannot hold.)*
- **Where is masking enforced?** Does the **Flight layer apply role-aware column
  masking + row filtering server-side**, returning already-(un)masked Arrow per
  the caller's grants — or does OliverDB return raw Arrow and the app masks?
  *(App-side masking breaks the "masking at the query layer, not the UI" claim —
  raw PII/PCI would already be in the app process.)*
- **RBAC tag schema:** what tags does OliverDB attach, and how does our
  fine-grained RBAC consume them (row-level, field-level)? Fail-closed on a
  missing tag?

### D. Schema & denormalization ownership
- Who denormalizes **business dimensions** (`agent_id`, `store_id`, `owner_team`,
  `cost_center`) onto telemetry rows — OliverDB at ingest (from resource attrs),
  or us? *(Decides AD-010 / reconciliation B5.)*
- Who owns the **OTLP → columnar schema** mapping and the `gen_ai.*` attribute
  extraction?

### E. Durability, freshness, retention, late spans
- **Durability:** Arrow is in-memory by nature — confirm on-disk persistence
  (Arrow IPC/Parquet) **+ the durable WAL** (SRS §2.1) so acked telemetry survives
  a worker restart.
- **Freshness SLA:** span emitted → queryable (drives 5-min sweepers + alerting).
- **Retention / sampling / down-sampling:** who enforces, and the config
  interface (competitive gap AD-012 item 8).
- **Late-span visibility:** how quickly is a late span **queryable to a re-read**
  (ingest latency + compaction)? *(Bounds our overlap/trailing-lag sizing.)*
- **Cross-signal correlation:** are spans/metrics/events/logs joinable by
  `trace_id`/`span_id` server-side, or do we correlate?

### F. Scale / HA / DR
- **Multi-region / DR** replication (Enterprise tier) for OliverDB + Cloud SQL.
- Backpressure / dedup / exactly-once at peak volume.

## 4. Sweeper cursor & completeness design (ours; partly depends on §3.B)

Two *different* completeness problems:
- **Read completeness** (every span read exactly once) — solved cleanly by a
  **commit-ordered `long` seq** cursor (`WHERE seq > last_seq`). If seq is
  allocation-ordered, add a **committed-watermark** or a **trailing seq-gap
  re-scan**. If only an ingest timestamp is available, use a **composite
  `(ingest_time, span_id)` cursor + trailing safety lag δ + overlap window**.
  *(Event time is never a safe cursor — unbounded lateness.)*
- **Trace / aggregate completeness** (all of a trace's spans present when we
  aggregate it) — **not** solved by any cursor, because a trace's spans arrive
  across time (slow tools, distributed sub-agents). Handle with:
  - **grace period** before a trace is "final" (root `end_time` + grace, or
    "no new spans for N");
  - **idempotent UPSERT** aggregation (already AD-006) so late spans/double-reads
    **converge** the result; and
  - a **periodic wide reconciliation** sweep for long-tail stragglers.

**Shape:** cursor (seq preferred) + trailing guard + overlap for boundary races +
grace + idempotent UPSERT + reconciliation. Overlap handles *seconds*; UPSERT +
reconciliation handle *minutes*. Tune overlap/lag to **measured p99 lateness**,
not a guess.

## 5. Platform gaps (ours — beyond the 3 excluded features)

- **Inline-enforcement claim ⚠️:** the platform observes OTel **post-hoc**; it
  **cannot block an already-executed** LLM/tool call. REDACT can be enforced at
  ingest; WARN/audit are post-hoc; true **Block/Require-Approval** need an
  enforcement point in the agent path (proxy/SDK) — not in this architecture
  (AD-012). → adjust the proposal's "in-line Block" language (see §6).
- **Instrumentation is a customer dependency, unassisted** (proposal §6.1); V1
  assumes agents **emit** `gen_ai.agent.id` (AD-001a) with **no resolve-by-name
  fallback**. If Wendy's agents don't emit `gen_ai.*` (agent_id, session_id,
  **usage tokens**, RETRIEVAL spans), inventory/session/RAG/**cost** degrade
  silently. **Cost depends on emitted tokens.** #1 POC-success risk.
- **Drift detection** — not specced (AD-012 item 6).
- **Experimentation / regression / datasets** — not specced (AD-012 item 3).
- **Health-status thresholds + SLOs** — deferred (SRS §7.3/§4.4); need config.
- **Proposal-introduced, no requirement behind them:** TOP Skill **MCP server** /
  dual access plane; **Adaptive UI Synthesis**; **HA/DR/multi-region** design.

## 6. Proposal claims contingent on partner answers (edit list)

| Proposal claim | Depends on | If not available → adjust |
| :-- | :-- | :-- |
| §2.2 "masking enforced at the **query layer, not UI**" | §3.C server-side Flight masking + reversible redaction | Soften to app-side/limited masking, or scope to what OliverDB enforces |
| §2.2 field-level **unmask** with permission | §3.C redaction **reversibility** | Remove unmask claim if redaction is destructive |
| §2.3 / §1.1 governance **"enforced in-line … Block"** | Enforcement point in agent path (none today) | Reword to "detect + record + redact-at-ingest; Warn/audit post-hoc"; drop "inline Block" |
| §3.3 "**ingest and index once**, no double-index tax" | §3.A OliverDB indexing for our access patterns | Confirm indexes exist for `trace_id` + time/agent scans; else caveat |
| §2.4 tamper-evidence / audit hash-chain | Cloud SQL audit store + append-only controls | Confirm implementation owner; already honestly scoped |
| §2.5 "**GenAI semconv native**" end-to-end | Wendy's agents emit `gen_ai.*` | Add instrumentation-coverage caveat; pull resolve-by-name forward |
| §5 metering "active **agent instances** / tokens / forecasting" | Instances discoverable + tokens emitted | Caveat cost/forecast on token emission; instances on `service_instance_id` |
| §1.1 external integrations / alert routing | AD-014 Feature C (excluded) | Mark alerting as a build item, not shipped |

## 7. Open reconciliation decisions (still to close)

- **B1 storage** — *resolved* by the two-store split (§2).
- **B2** status model, **B3** discovery timing (ingest vs sweeper), **B4**
  model/tool/system as tables vs edges, **B5** metadata source (resource-attrs vs
  registered / who denormalizes — overlaps §3.D). Still open.

## 8. Reminder — excluded from "assume implemented"

The three AD-014 pre-demo features are **not** built in this baseline: Failure
Spread (code-vs-deployment), Security Evaluations, Alerting. Any gap that a demo
would expose in these is expected, not a defect.
