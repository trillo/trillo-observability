# Trillo Neoclouds & Private AI Cloud Observability — Build Plan (Demo)

**Status:** the **how / when** for the demo build. Companion to the **PRD** (`...-PRD.md`, *why / scope*) and
the **Requirements** (`...-POC-Requirements.md`, *what / scenarios / acceptance*). This plan holds the
detailed data-model deltas, the multi-tenant conversion, the topology/allocation model, the simulator
invariants, the deployment-readiness tiers, and the **ordered build slices** the team develops incrementally
with the **Trillo AOS Claude Code plugin**. App: `NeoCloudObservability` (`.trillo/577`). Last updated
2026-09-05.

---

## 1. Doc roles & demo-first scope
- **PRD** = why/scope; **Requirements** = what/scenarios; **this Plan** = how/when + detailed model + slices.
- **Demo build scope is unified** — the demo shows the full vision on **synthetic data** (former V1 *and* V2).
- The V1/V2 split is retained only as a **deployment-readiness tier** (below), not a build gate.
- **Connectors are partner-gated:** real ingestion from a live neocloud is a big effort deferred until a
  neocloud partner engages. The **simulator stands in for the connectors** by emitting data in the shape the
  real connectors would produce — so the swap is later a connector change, not a remodel.

## 2. Locked design decisions
1. **Asymmetric tenancy.** Operator = **tenant-0** (sees all). **Tenant-scoped classes** — `TenantProfile`
   (the **1:1 sidecar** of the framework `Tenant`, **keyed by `tenantId`**), `Job`, `Allocation`,
   `TenantUsage` — carry `tenantId` (RLS). The **shared fleet + telemetry + rollups + findings-about-shared-
   infra are operator-global** (no `tenantId`). AOS applies tenancy **per-class by presence of `tenantId`**.
   **Application classes carry `tenantId`, never a `tenantProfileId`;** profile attributes come from a
   `TenantProfile` join on `tenantId` (the sidecar pattern used to extend `User` elsewhere). A tenant's fleet
   view is **derived by joining through `Allocation`**.
2. **Flip `AppConfig.multiTenant` now** — it's immutable after first tenant registration, so convert before
   seeding demo data (operator = tenant-0; customers = tenants 1..N).
3. **Demo tenant perspective = "view as tenant" impersonation** from the operator console, with real RLS
   behind it. No separate tenant-login theater.
4. **Topology vs Allocation are two time-aware graphs, modeled differently:**
   - *Physical topology* (near-static): normalized entities + **edge validity** (`validFrom`/`validTo`). No
     `Topology`-snapshot class. "Topology as of T" = query.
   - *Allocation* (fast-changing): time-interval rows. "Allocation as of T" = query. Management plane is the
     **trusted source**.
5. **Topology is a set of typed relationships between components.** The typed entities (`Node`, `Gpu`,
   `FabricSwitch`, `StorageSystem`, NVLink domain, rack) are the components; **`ComponentRelation`** relates
   two of them directly — `(fromType,fromRef)`↔`(toType,toRef)` with a `kind` (`contains` / `nvlink` /
   `compute_fabric` / …), validity, and an optional `linkId` → `FabricLink`. "What is X related to" = query
   relations on either endpoint. **One physical topology** (built from inventory); **many logical topologies**
   (one per job-run, derived), materialized as **`AllocationMember`** (job-run ↔ component,
   endpoint/traversed). **Root cause = the lowest common denominator (LCA)** over the relations / memberships.
6. **Shared-resource blast radius.** Shared components (switch / rack / cooling / power / storage) serve many
   tenants; blast radius **fans out through topology** (switch → `ComponentRelation` → node → GPU →
   allocations at T → tenants), classified **`direct` / `degraded` / `potential`** by failure-domain type.
7. **Telemetry is operator-only.** No `tenantId` on `OtlpTelemetry`; a denormalized `tenantSlug`/
   `tenantProfileId` hint may be stamped from the allocation, but scoping is derived, not enforced. "Trust
   the management plane, flag mismatches" — telemetry on a GPU with no active allocation → inventory-gap
   finding.
8. **Store: Postgres-first, OliverDB behind a config switch** (integration reused from Agent Observability).
9. **Cut line** — build the money-shot core first (Slices 0–8); the folded-in breadth (Slice 9) is cut first
   if the 3–4-week timeline slips.

## 3. Data-model deltas

**Add**
| Entity | Purpose | Tenant-scoped? |
|---|---|---|
| `ComponentRelation` | typed M:N relationship between two components: `(fromType,fromRef)`↔`(toType,toRef)`, `kind`, validity, optional `linkId`→`FabricLink` | no (operator-global) |
| `AllocationMember` | job-run ↔ component membership (endpoint/traversed) — the materialized logical topology | no (operator-global; refTenantId) |
| `Report` | saved analytics report (`generatedBy` ai/ml/sql/rule, scope, period, sections) | no (derived visibility) |
| `StorageSystem` | shared FS / object / NVMe (`kind`, `role`, throughput/iops rated, `shared`) | no |
| `StorageMount` *(optional)* | `storageSystemId → nodeId` for storage blast-radius fan-out | no |
| `Forecast` / `PredictionModel` | RUL / anomaly forecast output (prediction beat) | no |
| `RemediationAction` | actuation **preview** record (dry-run: action, target, blast-radius check, status) | no |

**Modify**
- **Replace `tenantProfileId` with `tenantId`** across all classes (`Job`, `Allocation`, `TenantUsage`, the
  `OtlpTelemetry` denormalized hint, `SimulationScenario.targetTenantId`). `Job` / `Allocation` /
  `TenantUsage` get `tenantId` as the RLS key. **`TenantProfile` becomes the 1:1 sidecar keyed by `tenantId`**
  (repurpose its `aosTenantId` → a required, unique `tenantId`); no `tenantProfileId` anywhere.
- **`ComponentRelation`** carries validity (`validFrom`/`validTo`) so topology is queryable as-of-T.
- **Do not** add `Node.fabricSwitchId` (superseded by `ComponentRelation`).
- **`OtlpTelemetry`**: add `storage` to the `source` enum + a `storageSystemId` flattened column (it already
  has `synthetic`, `executionId`, and the fleet semconv columns).
- **`SimulationScenario`**: add a `shared_fs_contention` type (distinct from node-local `storage_starvation`).
- `Gpu.nvlinkDomainId` stays a string ref (NVLink domain is intra-rack; no class needed for the demo).

## 4. Simulator invariants (physical consistency — enforce or the demo loses credibility)
`generate_synthetic_fleet` must satisfy these, and `validate_topology` should fail the seed if any are
violated:
- GPU → exactly one Node; Node → exactly one Rack.
- GPU → at most one NVLink domain; **an NVLink domain never spans racks**.
- **Leaf** links stay within a rack; only **spine** links cross racks; a node's NVLink/leaf neighbors are
  all same-rack.
- A GPU is **exclusively allocated** at any instant (MIG is the one documented exception) — no overlapping
  `Allocation` intervals per GPU.
- **Shared components (switch, rack, storage) may serve many tenants concurrently** — expected, and is what
  makes the shared-blast-radius beat work.
- Allocation windows sit within `[job.start, job.end]`; the bound GPU exists and is active during the window.
- Every `SimulationScenario` targets a consistent entity and produces the expected detector finding.

## 5. Deployment-readiness tiers (all shown in the demo)
| Tier | Meaning | Capabilities |
|---|---|---|
| **Day-one (provider-native)** | deployable with no tenant cooperation | topology, failure detectors, blast radius (direct + shared), real utilization / MFU, scoped chat, alerting, exec dashboard |
| **Needs a connector** | real data needs a specific integration | framework tier (SDC confirm, NCCL patient-zero, perf-Q&A), deep fabric (PFC/RoCE link-level), **storage** (WEKA/VAST/Lustre) |
| **Needs maturity** | product/POC-validation before real use | prediction, **actuation** (trust ladder — demo shows preview only), billing/chargeback, tenant-facing login |

## 6. Build slices (ordered; each = entities + functions + agents + acceptance)
Develop incrementally with the Claude Code plugin. **Slices 0–8 = core demo (cut line); Slice 9 = folded-in
breadth**, cut first if time slips.

- **Slice 0 — Tenancy + model deltas.** Flip `multiTenant`; add `tenantId` to `Job`/`Allocation`/
  `TenantUsage`; add `ComponentRelation` + `AllocationMember`, `Report`, `StorageSystem`; `OtlpTelemetry` storage
  source/column. *Accept:* app deploys multi-tenant; operator=tenant-0; RLS on the three classes verified.
- **Slice 1 — Simulator.** `generate_synthetic_fleet` seeds operator + tenants, consistent topology,
  `Allocation` intervals, `SimulationScenario` timelines; emits **real-connector-shaped** rows through
  `ingest_otlp_telemetry` / `ingest_scheduler_events`; `validate_topology` passes. *Accept:* invariants hold;
  data flows the real ingest path.
- **Slice 2 — Topology & Fleet Map.** `get_topology_as_of`, `get_fleet_topology`, `get_node_detail`. *Accept:*
  live map renders region→GPU + fabric with health/occupancy.
- **Slice 3 — Detectors & Reliability.** `run_detectors`, `get_finding_evidence`, `search_findings`,
  `correlate_findings`; cordon-recommend. *Accept:* ECC/row-remap and thermal-throttle findings with evidence.
- **Slice 4 — Allocation, Logical Topology & Blast Radius.** `resolve_logical_topology` (project allocation →
  `AllocationMember`), `get_allocations_as_of`, `resolve_blast_radius` as an **LCA over
  `AllocationMember`** (as-of-T + shared fan-out + impact class), **Allocation Timeline** view. *Accept:*
  a shared-switch failure names all affected tenants at the event time, and Jobs 1/7/12 resolve to a single
  common root.
- **Slice 5 — Utilization illusion (money shot).** `rollup_utilization`, `get_utilization_summary` (MFU vs
  `GPU_UTIL`, reclaimable idle by tenant/cluster). *Accept:* "GPU_UTIL 91% / MFU 27%, 40 reclaimable GPUs."
- **Slice 6 — SRE agent + Ask the Fleet.** Scoped, rehearsed Q&A over V1 data via functions-as-tools.
  *Accept:* answers the demo question set with drill-down.
- **Slice 7 — Alerting.** `generate_alerts` (dedup, blast-radius context), `deliver_alert`,
  `acknowledge_alert`, `test_alert_channel`. *Accept:* one alert per root cause, routed with blast radius.
- **Slice 8 — Executive Dashboard.** `get_executive_summary`, `estimate_capacity`. *Accept:* one-screen P&L +
  risk, every number drills down.
- **Slice 9 — Folded-in breadth (cut line).** view-as-tenant (`resolve_tenant_view` via allocation join);
  `shared_fs_contention` storage beat; `forecast` (prediction); deep-fabric view; billing/chargeback view on
  `TenantUsage`; `preview_remediation` (actuation dry-run + blast-radius check). *Accept:* each beat renders on
  synthetic data; readiness tier labeled.

Background: `run_full_sweep`, `rollup_metrics`, `refresh_baselines`, `reconcile_topology` run on the AOS
scheduler.

## 7. Functions & agents catalog (the addendum)
Existing (generated) functions to keep/extend: `ingest_otlp_telemetry`, `ingest_scheduler_events`,
`generate_synthetic_fleet`, `run_detectors`, `resolve_blast_radius`, `rollup_utilization`, `rollup_metrics`,
`correlate_findings`, `generate_alerts`, `deliver_alert`, `acknowledge_alert`, `test_alert_channel`,
`get_fleet_topology`, `get_node_detail`, `get_utilization_summary`, `get_executive_summary`,
`get_finding_evidence`, `get_entity_activity`, `query_fleet_health`, `search_findings`, `estimate_capacity`,
`reconcile_topology`, `refresh_baselines`, `run_full_sweep`, `save_ai_analysis`.

New functions to generate (with slice): `build_physical_topology` (inventory → `ComponentRelation`, 1/2),
`get_topology_as_of` (2), `resolve_logical_topology` (project allocation → `AllocationMember`, 4),
`get_allocations_as_of` (4), `validate_topology` (1), `resolve_tenant_view` (9), `forecast` (9),
`preview_remediation` (9), `generate_report` (8/9), storage detectors folded into `run_detectors` (9). Extend
`resolve_blast_radius` to be an **LCA over `AllocationMember`** (shared fan-out + as-of-T, 4); extend
`generate_synthetic_fleet` for tenancy + invariants + scenarios (1).

**Authoring-heavy specs** for `build_physical_topology`, `resolve_logical_topology`, and the **LCA**
`resolve_blast_radius` are in `Trillo-Neoclouds-Observability-Function-Specs.md`. **Slice-0 rename impact:**
16 existing functions reference the old `tenantProfileId` — **regenerate the function + UI layer via the
plugin** from the updated entities + that spec (mapping: tenant-scoped classes → `tenantId`, operator-global
→ `refTenantId`; `TenantProfile` joined by `tenantId`), rather than hand-patching.

Agents: the **SRE agent** (scoped Q&A, Slice 6) grounded via the read functions as tools; a background
**analysis agent** (optional) that attaches `AiAnalysis` to findings.

## 8. Risks / open items
- **Timeline:** 3–4 weeks / 2–3 people holds only with the Slice 0–8 cut line; Slice 9 is breadth, not depth.
- **Connectors:** deferred to a partner engagement; the readiness tier + live-vs-synthetic map keep prospects
  honest.
- **Actuation:** demo shows **preview only**; real deployments stay recommend-only until POC-validated.
- **OliverDB switch:** validate the reused integration flips cleanly at demo scale before relying on it.
