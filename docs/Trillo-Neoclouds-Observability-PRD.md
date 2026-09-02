# Trillo Neoclouds Observability — Product Requirements (V1)

**Status:** Draft for review — scopes V1 (and the V2 roadmap) for Trillo Neoclouds / Private AI Cloud
Observability. Reconciles with `Trillo-Neoclouds-Observability-Brochure.md` (the brochure is the vision /
roadmap; **this PRD is the source of truth for V1**). Companion detector catalog: the AI-infrastructure
root-cause matrix (25 failure modes tagged by telemetry source) is the empirical basis for §4.

**Goal:** Ship a V1 with *high value to the customer* — more features in V2. V1 wins the **NeoCloud
provider** with reliability + blast radius + real utilization + **job correlation**, on a data model built
to extend to their **tenants** next, and to the **Private Cloud** edition with no fork.

> **Next artifact:** after this PRD is agreed and the **Cursor / Aurora** pain-point conversations land, the
> detailed spec goes into a `...POC_Requirements_Final`-style document (functional + data + UI screens +
> acceptance) plus a **telemetry-simulator requirements** doc (V1 demos on synthetic data). This PRD is the
> scope those inherit.

> **V2 in one line (locked):** V2 = **advanced analytics (prediction) · actuation · NeoCloud-customer
> (tenant) views · billing**. Everything else — including **job correlation** — is in V1's sights.

---

## 1. Strategy in one page

- **The moat is topology + correlation.** Per the component glossary's own thesis — *"GPU performance
  depends on the entire pipeline, not just the GPU"* — the product connects a failure or slowdown down
  through scheduler → GPU → NVLink → fabric → storage → power, **and up to the tenant**. Everyone can draw
  a GPU dashboard; almost no one does the correlation. That is the spine, not a feature.

- **Target one audience, model for both.** V1 ships the **operator (provider)** product/UI. The data model
  is **tenant-typed and RLS-scoped from day one** — which costs nothing extra because the tenant/job
  dimension is *required* for V1's best feature (blast radius) anyway. V2 adds tenant-facing views on the
  **same graph**, no rewrite.

- **Provider-first, then extend to their customers.** The provider is the buyer (they pay, they deploy it
  in their own cloud). V1 excites them with *their* P&L — fewer tenant-visible incidents and less wasted
  capacity. V2 hands them tenant-facing job/hardware visibility they can *offer to their customers* as a
  differentiator.

- **One code base, two editions.** NeoCloud and Private AI Cloud observe *identical* infrastructure. They
  differ only in narrow, configurable ways — tenant type, billing mode, emphasis, deployment. **One data
  model, one code base, two editions** (§9). The `tenant` abstraction is the keystone for all three splits:
  operator/tenant persona, NeoCloud/Private-Cloud edition, and blast-radius correlation.

- **The telemetry seam = the product seam.** Provider-native telemetry (DCGM, node, kernel, scheduler,
  fabric) deploys fleet-wide with **zero tenant cooperation** → **V1**. Framework instrumentation (Ray /
  PyTorch / vLLM) lives *inside the tenant's container* → **V2 tenant tier**. Same line as provider→tenant.

- **V1 store: Postgres now, OliverDB integrated.** V1 implements against **PostgreSQL** as the target store
  with **OliverDB integration**. At real fleet scale OliverDB carries the hot telemetry path (Postgres
  alone won't hold fleet-wide cardinality); for V1 + synthetic-data demo, Postgres target + OliverDB
  integration is the pragmatic path. Crossover to OliverDB-as-hot-store is marked in §8.

---

## 2. Personas, editions & tenancy

| | Primary V1 user | V2 extension |
|---|---|---|
| **NeoCloud edition** | Provider ops / SRE / platform / capacity & FinOps | Provider's external customers (tenants) |
| **Private Cloud edition** | Enterprise infra / platform / ML-platform team | Internal teams / cost-centers |

**Personas.** *Operator* (V1): fleet health, blast radius, utilization/margin, capacity. *Tenant* (V2):
"is my job healthy? is their hardware hurting me?" Two lenses on one graph.

**Tenancy model (AOS-native).** The observability product is a **multi-tenant AOS app**. The neocloud's
**customer = an AOS `Tenant`** (system class); its **`tenantId` drives RLS isolation**. We extend it with a
**`TenantProfile`** class carrying observability attributes — tenant **type** (external-customer |
internal-team), **billing mode** (invoice | chargeback/showback), SLA terms, contacts. Every graph node and
signal is tenant-scoped, so V2 tenant views become "the customer logs into their RLS-scoped slice" with no
model change. The **edition delta** (NeoCloud → Private Cloud) is entirely `TenantProfile` type + billing
mode + emphasis + deployment — a config, not a schema change.

---

## 3. V1 scope — the six customer priorities, mapped

| # | Priority | V1 | V2 |
|---|---|---|---|
| 1 | **Topology** | Compute inventory (GPU/node) auto-discovered; physical hierarchy (rack/AZ/region); NVLink domains; **fabric switches + links** in the graph (light ingestion) | Deep fabric analytics (link-level IB/RoCE, bisection BW, oversubscription, rail contention) |
| 2 | **Failure signals** | Full provider-native detector set (§4) — hardware degradation, XID, thermal/power throttle, NVSwitch/Fabric-Manager, starvation, NUMA, clock drift, scheduler pathologies | Framework-confirmed SDC, NCCL patient-zero, deep-fabric packet drops |
| 3 | **Utilization signals** | **Real** occupancy/idle — MFU vs the `GPU_UTIL` lie — at fleet & tenant level | **Billing** (per-tenant metering→invoice/chargeback), margin, oversubscription tuning, capacity forecast |
| 4 | **Answer performance questions** | *Scoped* chat over V1 data ("which nodes are throttling?", "idle GPUs by tenant") | Full perf-Q&A copilot — "why is my job slow: GPU, fabric, storage, or contention?" (framework tier) |
| 5 | **Predictive failure alerts** | *Degradation*-cordon recommend (reactive — e.g., row-remap exhaustion imminent, thermal creep). **Not** forecasting | **Prediction / advanced analytics** — anomaly & RUL models on V1-collected history |
| 6 | **Job correlation (Slurm/K8s)** | **Full — both schedulers.** K8s (inference) + Slurm (training) → **blast radius**: DCGM anomaly → job → tenant, **no framework needed** | In-job view (loss/gradient/step timing) via framework tier |

**Schedulers:** V1 covers **both K8s (inference) and Slurm (training)** — the partner runs both, and job
correlation needs both. **Implementation sequences Slurm first.**

**V1 headline outcomes.** (a) *Catch the bad hardware before it kills a tenant's job, and name the exact
blast radius.* (b) *Show real utilization — "half your paid-for fleet is idle" — the margin moment.* Both
provider-native, zero tenant cooperation.

---

## 4. V1 ingestion scope — by telemetry source (the detector catalog)

V1 ingests **only provider-native sources** — each deployable fleet-wide without touching a tenant's
container. Detectors below are the V1 subset of the root-cause matrix.

**V1 sources:** `[DCGM/NVML]` · `[Node Exporter]` · `[Host Syslog / Kernel]` · `[Scheduler API — K8s + Slurm]`
· `[Fabric Manager]` · `[Switch/Fabric — inventory, links, basic counters]`

| Detector | Source(s) | V1 |
|---|---|---|
| Double-bit ECC / HBM row-remap exhaustion (1.1) | DCGM | ✅ degradation-cordon recommend (reactive) |
| GPU off the bus / XID 79 (1.2) | Kernel/Syslog | ✅ |
| Silent Data Corruption (1.3) | DCGM (suspect) → Framework (confirm) | ⚠️ **V1 flags suspect device**; V2 confirms via loss correlation |
| NVSwitch / Fabric-Manager faults (1.4) | Fabric Mgr + DCGM | ✅ |
| Silent thermal throttling (1.5) | DCGM | ✅ |
| Thermally-induced power capping (1.6) | DCGM | ✅ |
| GPU starvation / **utilization illusion** (2.2) | DCGM + Node | ✅ (VRAM-full/SM-idle from DCGM; IOPS from Node) |
| Zombie actors / VRAM leak (2.3) | Scheduler + DCGM | ✅ |
| Sub-optimal NUMA mapping (3.3) | Node + DCGM | ✅ |
| PTP/NTP clock drift (3.4) | Node | ✅ |
| Gang-scheduling deadlock (4.1), fragmentation, preemption loops | Scheduler (K8s + Slurm) | ✅ |
| NCCL/RCCL ring deadlock — patient-zero (2.1) | Framework + Switch | ❌ V2 |
| Lossless packet drops / PFC / RoCE (3.1) | Switch/Fabric (deep) | ❌ V2 |
| Asymmetric interconnect latency (3.2) | Switch + Framework | ❌ V2 |

**Fabric (priority 1):** V1 includes fabric **switches and links** in the graph with **light ingestion**
(status + basic counters) — enough to render topology and place blast radius. Deep per-link analytics
(PFC/RoCE drops, bisection BW) are V2.

**Facility / BMS (power, cooling):** **not** a V1 integration. V1 **infers** facility problems from
*simultaneous, localized multi-node thermal anomalies*. Deep BMS integration is V2.

**Ingestion path** (matches brochure): standard exporters → **OTLP** → **PostgreSQL (V1 target) + OliverDB
(integrated engine)**; non-standard metrics mapped at ingestion (no re-instrumentation). **No proprietary
agents.**

---

## 5. Data model — topology graph, tenant-typed, RLS

One entity graph on Trillo AOS (multi-tenancy + RBAC/RLS out of the box), telemetry via Postgres→OliverDB.

- **Physical / compute:** `Region · AZ · Rack · Node · GPU` (+ `HBM`, `NIC`, `NVLink/NVSwitch` domain).
- **Fabric:** `FabricSwitch` (leaf/spine/core), `FabricLink` — **in the V1 graph** (inventory + light
  telemetry); deep analytics V2.
- **Tenant / workload overlay:**
  - **`Tenant`** = AOS **system class**; `tenantId` → **RLS**. *Not* redefined here.
  - **`TenantProfile`** = our extension: tenant **type** (external-customer | internal-team), **billing
    mode**, SLA, contacts.
  - **`Job`** (K8s + Slurm), **`Allocation`** (job → node/GPU). Powers **blast radius** and is the seam to
    the V2 tenant edition.
- **Signals:** detector outputs as `Finding` / `Alert` linked to the graph node they concern, tenant-scoped.
- **Discovery + reconcile:** telemetry-first auto-discovery (DCGM/node/scheduler) + a background
  **reconcile sweeper** against authoritative sources (K8s + Slurm APIs; Redfish/CMDB for physical). Fabric
  authoritative source (OpenSM/ibnetdiscover) is V2; V1 fabric = inventory from config/labels + light
  counters.
- **Isolation from day one:** every node and signal is `tenantId`-scoped (RLS) and RBAC-gated, so V2 tenant
  views expose only a tenant's own slice with no model change.

---

## 6. UI — three layers (flows to refine mid-build)

Organizing axis = the **operator's jobs-to-be-done**, not entity tables. Front door = answers; data-model
drill-down is the escape hatch at the bottom, not the entry.

- **Answer layer (front door).** **Highlight/triage cards** — "what needs attention now" — are the spine. A
  **scoped chat** rides along, answering over *V1 data only* (throttling nodes, idle GPUs by tenant, blast
  radius of node X). Not the deep perf-Q&A copilot (V2, framework tier).
- **Map layer (the spine of the UI).** The **live fleet/topology map** is home. Failures light up on it,
  utilization heat-maps onto it, **blast radius highlights** on it, jobs/tenants overlay on it.
- **Evidence layer (on demand).** **Meaningful dashboards** — GPU/node/fabric detail, raw DCGM Xid/ECC
  counters — reached *from* a card or map node to verify source of truth. Same drill-down as the Agent UI,
  but not the front door.

**Discipline:** one surface **per workflow**, not per entity. V1 workflows: *triage / situational awareness
· investigate a failure (→ blast radius → recommend) · reclaim capacity · per-tenant view.*

**Synthetic data → complete model on the UI.** V1 demos on **synthetic data**, so the UI can assume the
**complete data model** and render the full picture (fabric links, tenant overlays, V2-shaped surfaces)
without waiting for live ingestion of every source. Discipline: keep an internal **live-vs-synthetic map**
so we never *claim* live capability V1 hasn't built.

**Mid-build checkpoint:** review UI flows after the map + one full workflow (investigate-a-failure) are
wired, before building the rest.

---

## 7. Alerting & remediation posture (V1 = read-mostly)

- **Alerting:** de-duplicated, **blast-radius** alerts. Channels **same as Trillo Agent Observability**
  (Slack / Teams / PagerDuty / webhook). Rules configurable (SLO, thresholds, severity).
- **Remediation — V1 recommends, does not act.** V1 = **detect → attribute blast radius → recommend →
  alert.** No writes into the provider's control plane. **Actuation (drain / cordon / quarantine / burn-in)
  is V2**, delivered as an opt-in, POC-validated **trust ladder** (§9). Rationale: V1 earns trust in the
  *detection* — no operator lets us drain nodes until they trust our "this GPU is bad" call — and V1
  read-only already delivers the high-value story.

---

## 8. Non-functionals

- **Deployment:** BYOC — collectors + PostgreSQL + OliverDB run **in the provider's own environment**;
  fleet & tenant data never leave it (brochure promise).
- **Target store:** **V1 = PostgreSQL target + OliverDB integrated.** The OliverDB integration is **already
  designed in Trillo Agent Observability and reusable here**, so the switch is essentially a **config /
  env-variable change** (plus minor wiring), not a rebuild. **Crossover:** once fleet-wide telemetry
  cardinality exceeds what Postgres holds economically, flip to OliverDB as the **primary hot telemetry
  store**; Postgres retains metadata/policy. (Same split the Agent product uses.)
- **Security:** in-environment, provider- and tenant-owned data, encryption, **RBAC + RLS per-tenant
  isolation**, versioned policy, complete audit trail. Actuation (V2) is a gated, high-`effect`,
  `sensitive` capability — audited, dry-runnable.
- **Scale / retention / latency targets:** *left open — to be worked during implementation / with the design
  partner.* No placeholder numbers committed here.

---

## 9. V2 roadmap — four headline buckets

**V2 = advanced analytics (prediction) · actuation · NeoCloud-customer (tenant) views · billing.**

1. **Advanced analytics (prediction).** Anomaly & RUL models on V1-collected history; framework-confirmed
   SDC; NCCL patient-zero; deep-fabric analytics (link-level IB/RoCE, PFC/RoCE drops, bisection BW,
   oversubscription); facility/BMS deep integration; the full **perf-Q&A copilot** (priority 4).
2. **Actuation (trust ladder).** (1) recommend [V1] → (2) assisted/one-click, dry-run + reversible +
   blast-radius-checked → (3) policy-based within operator guardrails → (4) closed-loop for the most mature
   operators. Each rung POC-validated in staging/canary before prod; per-operator opt-in;
   RBAC/capability-gated.
3. **NeoCloud-customer (tenant) views.** The tenant-facing edition on the same graph — the customer logs
   into their RLS-scoped slice; job/hardware health for *their* workloads; the **framework tier** (Ray /
   PyTorch / vLLM instrumentation) that unlocks in-job loss/gradient/step insight.
4. **Billing.** Per-tenant metering → invoice (NeoCloud) / chargeback-showback (Private Cloud); SLA
   reporting; reconciled, exportable GPU-hours.

**Private Cloud edition delta** rides on the same code base: internal-team tenant type, chargeback/showback
billing mode, on-prem packaging (§2).

---

## 10. Brochure reconciliation

The brochure describes the **mature** product; this PRD cuts V1 to the high-value core and roadmaps the
rest. The brochure is a living doc — anything V1 earns that's worth showing gets added *back* into it.

| Brochure claims | PRD placement |
|---|---|
| Fleet inventory & topology, GPU/node health, live fleet map, utilization | **V1** (fabric = switches + links, light) |
| Failure detection, blast radius, alerting | **V1** |
| Root cause (AI-assisted) | **V1 scoped chat** / **V2 full copilot** |
| Fabric health (NVLink + IB/RoCE throughput) | **V1** NVLink + links (light); **V2** deep IB/RoCE |
| Per-tenant metering & billing, capacity forecast, oversubscription | **V2 (billing bucket)** — V1 = light occupancy/idle |
| AI Investigation Copilot (MCP), specialized analysis, background sweepers | **V2** — V1 = scoped chat + basic findings |

**Action:** re-cut the brochure to **lead with the V1 reliability + correlation + real-utilization +
job-correlation story**, add the missing **full-stack correlation** headline, and present
billing/prediction/deep-fabric/actuation as the explicit **roadmap** — no hard "available today" claims a V1
POC can't back.

---

## 11. Open decisions (resolved)

1. **Design-partner profile — RESOLVED.** K8s (inference) + Slurm (training).
2. **Scheduler scope/ordering — RESOLVED.** Both in V1; **implement Slurm first.**
3. **Non-functional targets — DEFERRED.** Left open; to be worked. **Target = PostgreSQL + OliverDB
   integration.**
4. **Alerting channels — RESOLVED.** Same as Trillo Agent Observability (Slack / Teams / PagerDuty /
   webhook).
5. **V1 fabric depth — RESOLVED.** Include fabric **switches + links** in V1 with **light ingestion**.
6. **Actuation — CONFIRMED V2** (opt-in, POC-validated trust ladder). V1 read-mostly.

---

## 12. V1 definition of done (POC success)

The provider POC is a win when, in their own environment (or on synthetic data for the demo), Trillo can:

1. **Auto-discover** the fleet and render the live topology map — compute + physical + NVLink + fabric
   (switches + links) — with tenant/job overlay from **both K8s and Slurm**.
2. **Detect** a degrading/failed GPU or node from the §4 provider-native detectors, and **recommend cordon
   before a panic** on at least the ECC/row-remap and thermal-throttle paths.
3. **Attribute blast radius** — name the exact tenant(s)/job(s) a failing device affects — with no framework
   instrumentation, tenant-scoped by RLS.
4. **Surface real utilization** — MFU vs `GPU_UTIL` — and quantify reclaimable idle capacity per tenant and
   cluster.
5. **Alert** with de-duplicated, blast-radius-aware routing to the configured channels.
6. **Answer** basic fleet questions via the scoped chat, and let the operator **drill to evidence** to
   verify any finding.
