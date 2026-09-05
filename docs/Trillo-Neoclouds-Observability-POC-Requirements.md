# Trillo Neoclouds & Private AI Cloud Observability

## Proof of Concept Requirements and Demonstration Specification

**Status:** Detailed build spec for the V1 POC. Companion to `Trillo-Neoclouds-Observability-PRD.md` (scope /
strategy — the source of truth for what V1 is) and `neoclouds-observability-pain-points.md` (the pain
catalog). The AI-infrastructure root-cause matrix (25 failure modes, telemetry-source-tagged) and
`neocloud_private_ai_cloud_glossary.md` are the empirical basis. **V1 scope agreed with the design-partner
team, 2026-09-03.**

---

## 1. Purpose

This document specifies the V1 proof-of-concept for Trillo Neoclouds / Private AI Cloud Observability: a
GPU-fleet observability product a NeoCloud provider (or enterprise private AI cloud) deploys **in their own
environment** to see fleet health, catch failures before tenants do, attribute blast radius to the exact
job and tenant, and surface real (not illusory) utilization — across the whole pipeline, not just the GPU.

The POC is demonstrated on **synthetic but realistic fleet telemetry** and is built on **Trillo AOS**
(model-driven, multi-tenant, RBAC/RLS, hot-deployable). It targets the **operator** persona in V1, on a data
model built to extend to the provider's **tenants** (V2) and to the **Private Cloud** edition with no fork.

**Demo-first scope (2026-09-05):** because the demo is the door-opener, the **demo build scope is unified** —
it shows the full vision on synthetic data, including former-V2 capabilities (prediction, tenant views,
billing / chargeback, deep fabric, and **actuation-as-preview**). The former V1/V2 split is retained as a
**deployment-readiness tier** (day-one provider-native · needs a connector · needs maturity), *not* a build
gate — see the PRD and the plan doc. On real fleets, remediation stays **recommend-only** until each
trust-ladder rung is POC-validated; the demo shows actuation as **dry-run preview** only.

---

## 2. POC Design Principles

### 2.1 User Experience First
The POC is judged on whether an operator can answer a real question — "what's wrong, who's hit, what's
wasted" — in a few clicks, not on raw metric coverage. Lead with answers; the data-model drill-down is the
escape hatch, not the front door.

### 2.2 One Connected Fleet
Topology is the spine. Every failure, utilization heat-map, and blast radius is projected onto **one live
fleet map**. The map is home; dashboards hang off it.

### 2.3 Drill Down, Not Search Again
From any answer (a card, an alert, a map node) the user drills to evidence — raw DCGM counters, the exact
job, the switch telemetry — without re-querying in a separate tool.

### 2.4 Synthetic but Realistic Fleet Data
The demo runs on a **fleet telemetry simulator** producing DCGM / node / switch / scheduler streams with
injected failures (SDC, Xid, thermal throttle, NCCL stall, fragmentation, storage starvation). Because it's
synthetic, the UI may assume the **complete data model** — but an internal **live-vs-synthetic map** keeps us
honest about what V1 ingests live.

### 2.5 Two Personas, One Graph (asymmetric tenancy)
The operator is **tenant-0** and sees the whole fleet. The neocloud's customers are **AOS tenants**, but
tenancy is *asymmetric*: the **tenant-scoped classes** — `TenantProfile` (the 1:1 sidecar of `Tenant`, keyed
by `tenantId`), `Job`, `Allocation`, `TenantUsage` — carry `tenantId` (RLS); the **shared fleet** (Region…GPU,
fabric, telemetry, rollups) is **operator-global** (no `tenantId`). **Classes reference `tenantId`, never a
`tenantProfileId`.** A tenant's fleet view is **derived by joining through `Allocation`**, not by owning fleet
rows.
The app is flipped to **`multiTenant` now** so the demo DB matches the final shape (see the plan doc). In the
demo, the tenant perspective is shown via **"view as tenant"** impersonation, with real RLS behind it.

### 2.6 Honest V1 Scope
V1 **recommends**, it does not act. No writes into the provider's control plane. Actuation is a V2 opt-in,
POC-validated trust ladder. V1's job is to earn trust in the *detection* first.

---

## 3. Core Information Model

The POC models the following as first-class entities or dimensions.

### 3.1 Region / AZ / Rack
The physical hierarchy. Example dimensions: `region` (us-west-2), `availabilityZone`, `rack` (r-14),
power/cooling domain, `failureDomain`.

### 3.2 Node
A physical compute node. Example dimensions: `nodeName`, rack, CPUs, RAM, NIC(s), NUMA layout, BMC address,
firmware/driver versions, `status` (healthy / degraded / drained / quarantined), first seen / last seen.

### 3.3 GPU
A single accelerator. Example dimensions: `gpuUuid`, `gpuIndex`, node, model (H100/GB200), HBM capacity,
NVLink domain, PCIe root, `health` (ok / degraded / failed), ECC/row-remap state, `status`.

### 3.4 Fabric & wiring
The scale-out network as inventory + light telemetry. Entities: `FabricSwitch` (leaf / spine / core),
`FabricLink` (endpoints, type IB / RoCE / NVLink, rail), and **`NodeToLink`** — the **many-to-many** join
letting a node fan out to multiple links, with a **`linkType`** discriminator (`compute_fabric` / `nvlink` /
`storage_fabric` / `management` / `tenant_network`), `rail`, and a validity interval. Shared blast radius
traverses **switch → `FabricLink` → `NodeToLink` → nodes → GPUs**. Deep per-link analytics are V2; V1 places
jobs and blast radius on the graph.

### 3.5 Tenant & TenantProfile (1:1 sidecar)
The neocloud's customer **is the AOS framework `Tenant`** (`tenantId` → RLS). **`TenantProfile` is a 1:1
*sidecar* of `Tenant`, keyed by `tenantId`** — it holds only the profile attributes the framework class
doesn't (type `external-customer | internal-team`, billing mode, SLA terms, contacts, `gpuHourlyRate`).
**Application classes carry `tenantId`, never a `tenantProfileId`;** profile attributes are resolved by
joining `TenantProfile` on `tenantId` when needed — the same sidecar pattern used to extend the `User` class
in another app. This is the keystone for persona (operator/tenant), edition (NeoCloud/Private-Cloud), and
blast radius.

### 3.6 Job & Allocation
`Job` = a scheduled workload from **K8s (inference)** or **Slurm (training)** — `jobId`, tenant, scheduler,
type, `status`, start/end, requested GPUs. `Allocation` = the job→node/GPU binding (the join that powers
blast radius).

### 3.7 Finding & Alert
`Finding` = a detector output (the failure/waste it found, the graph node it concerns, evidence, severity).
`Alert` = a routed, de-duplicated, blast-radius-aware notification derived from findings.

### 3.8 Correlation key — `executionId` / `jobRunId`
Infra telemetry arrives from multiple exporters (DCGM, node, switch, scheduler) that **do not share a trace
id**. A first-class **`executionId`** (a.k.a. `jobRunId`) column ties all telemetry for one job-run together
across sources — the same pattern Agent Observability uses to survive non-compliant / colliding trace ids.
See §11.2.2.

### 3.9 Topology vs Allocation — two time-aware graphs
Two graphs change at very different rates, so they are modeled differently (detail in the plan doc):
- **Physical topology** (`Node`, `FabricLink`, `NodeToLink`, `Gpu`, NVLink domain, switch, rack) — the
  *wiring*, near-static; the edges that can change carry `validFrom` / `validTo`. "Topology as of T" is a
  **query**, not a stored snapshot (no `Topology`-snapshot class).
- **Allocation overlay** (`Allocation`, time-windowed) — the *workload*, fast-changing; the management plane
  is the **trusted source**. "Allocation as of T" is a query over active intervals.
The two compose into the full graph at time T. **Shared components** (switch, rack, cooling / power domain,
storage) serve **many tenants at once**, so blast radius fans out through topology, not just direct GPU
bindings. A cheap by-product of trusting the management plane: telemetry on a GPU with **no active
allocation** surfaces as an inventory-gap / untenanted finding.

---

# 4. Scenario 1: Fleet Inventory & Topology

## 4.1 Demonstration Requirements
Show how an operator can:
- See the whole fleet — regions, racks, nodes, GPUs — discovered automatically.
- See how it's connected — NVLink domains and the IB/RoCE fabric.
- See health and occupancy at a glance, worst-case surfaced.
- Place any node/GPU on the map and see the jobs and tenants on it.

## 4.2 POC Requirements
The POC shall:
- Auto-discover the fleet from telemetry (DCGM / node / scheduler) and render a **live topology map** —
  region → AZ → rack → node → GPU, plus fabric switches and links.
- Overlay per-node/GPU health and real occupancy; surface the worst-case per rack/cluster.
- Overlay the tenant/job layer (from K8s + Slurm) — which job and tenant occupy which GPUs.
- Support filtering by region, rack, node, GPU model, health, tenant, job, and scheduler.
- Flag inventory gaps (a node emitting no telemetry; an unlabeled/untenanted allocation).

## 4.3 User Experience
**Screen:** `Fleet Map`

**Workflow:**
1. Operator opens **Fleet Map**; the default view is fleet health + occupancy by rack.
2. Hovering a rack shows nodes; a red node stands out.
3. Selecting the node reveals its 8 GPUs, NVLink domain, NICs, and the fabric links it rides.
4. A side panel lists the jobs and tenants currently allocated to that node.

## 4.4 Target Demonstration Outcome
The evaluator sees their entire fleet, correctly connected, in one view — and can go from "the fleet" to "a
specific GPU and who's on it" without touching a second tool.

---

# 5. Scenario 2: Failure Investigation

## 5.1 Demonstration Requirements
A GPU/node degrades or fails. Show how to:
- Locate the failing device.
- Determine root cause from provider-native signals.
- See the evidence (the actual counters/logs).
- Get a recommended action.

## 5.2 POC Requirements
The POC shall detect and surface the V1 provider-native detector set (per the root-cause matrix):
- **GPU/silicon:** double-bit ECC / HBM row-remap exhaustion, Xid 79 (off the bus), NVSwitch / Fabric-Manager
  faults, thermal throttling, power capping, SDC **suspect flag** (DCGM per-device anomaly; full confirmation
  is V2 framework tier).
- **Node/host:** storage-IO starvation, NUMA misalignment, PTP/NTP clock drift.
- **Scheduler:** gang-scheduling deadlock, fragmentation, preemption loops, zombie/VRAM-leak (scheduler ×
  DCGM).
For a selected failure, the POC shall show the **first/root signal**, the **evidence** (raw DCGM Xid/ECC
counters, kernel log line, throttle-reason bitmask), a **recommended action** (e.g. "cordon node r14-n03
before panic"), and a **historical trend** vs. baseline. **No actuation** — recommend only.

## 5.3 User Experience
**Screen:** `Reliability Explorer`

**Workflow:**
1. Operator filters findings to `severity ≥ high`.
2. Selects a degrading `r14-n03 / GPU-3` finding.
3. The panel shows: rising correctable-ECC trend crossing into row-remap exhaustion — **cordon before a
   double-bit panic**.
4. Evidence drawer shows the raw DCGM counters and the remap-availability curve.
5. **Impacted** panel (feeds Scenario 3):

| Entity | Impact | Evidence |
| :--- | :--- | :--- |
| `GPU-3 (r14-n03)` | Degrading | Row-remap availability 4% and falling |
| `job slurm-8821 (tenant Acme)` | At risk | Training job allocated to this GPU |
| `r14-n03` | Cordon recommended | Pre-empt double-bit panic |

## 5.4 Target Demonstration Outcome
The evaluator moves from "something's wrong" to a credible root cause + recommended action + evidence, on
provider-native telemetry alone — before the failure takes a tenant job down.

---

# 6. Scenario 3: Blast Radius & Job/Tenant Correlation

## 6.1 Demonstration Requirements
A device/link fails. Show — with no framework instrumentation — exactly which **jobs and tenants** are
affected, and how confident we are.

## 6.2 POC Requirements
The POC shall:
- Resolve **device → allocation → job → tenant** from the scheduler (K8s + Slurm) join.
- Resolve blast radius **at the time of the event (`as-of-T`)** from the time-windowed `Allocation` ledger —
  not just current bindings.
- For **shared components** (switch / rack / cooling / power / storage), **fan out through topology**
  (`NodeToLink`, rack membership, NVLink domain) to every tenant served — not only the directly-bound GPU.
- Classify each impacted job as **`direct` / `degraded` / `potential`** by failure-domain type.
- Provide an **Allocation Timeline** view ("who held what, when") as the operator's point-in-time lens over
  the fleet.
- Quantify blast radius (GPUs, jobs, tenants, GPU-hours at risk).
- Work for **both** K8s inference and Slurm training workloads.

## 6.3 User Experience
**Screen:** `Blast Radius` (opened from a finding or a map node)

**Workflow:**
1. From the GPU-3 finding, operator clicks **Blast Radius**.
2. The map highlights the affected node, its NVLink domain, and the fabric path.
3. A table lists impacted jobs and tenants:

| Job | Tenant | Scheduler | Impact | GPU-hours at risk |
| :--- | :--- | :--- | :--- | :--- |
| `slurm-8821` | Acme (external) | Slurm | Direct | 512 |
| `k8s/infer-42` | Acme (external) | K8s | Same rack, potential | 96 |
| `slurm-8830` | Globex (external) | Slurm | Same NVLink domain, potential | 128 |

## 6.4 Target Demonstration Outcome
The evaluator can answer "who does this failure hurt?" in one click — the multi-tenant capability, delivered
without any tenant-side instrumentation.

---

# 7. Scenario 4: Utilization & Waste (the utilization illusion)

## 7.1 Demonstration Requirements
Show the gap between what the dashboard *says* and what the fleet is *actually doing*, and quantify the
reclaimable waste — per tenant and per cluster.

## 7.2 POC Requirements
The POC shall:
- Compute **real utilization** (SM-active / SM-occupancy / tensor-active, and MFU where a workload FLOP
  estimate is available) alongside `GPU_UTIL`, exposing the **utilization illusion**.
- Surface **idle** and **allocated-but-unused** GPUs, with the reclaimable quantity per tenant and cluster.
- Detect **starvation** (VRAM-full / SM-idle — storage or fabric bound).
- Roll utilization up to fleet, cluster, and tenant level over time.

## 7.3 User Experience
**Screen:** `Utilization`

**Workflow:**
1. Operator opens **Utilization**; a headline reads "GPU_UTIL 91% · real MFU 27%".
2. A treemap shows reclaimable capacity by tenant; `Acme` has 40 GPUs allocated at <10% real use.
3. Drilling into one shows VRAM-full / SM-idle — a data-loader/storage bottleneck, not a GPU fault.

## 7.4 Target Demonstration Outcome
The "half your paid-for fleet is idle" moment — quantified, attributed, and provider-native.

---

# 8. Scenario 5: Ask the Fleet (scoped chat)

## 8.1 Demonstration Requirements
Let the operator ask questions in plain English and get answers grounded in V1 data — with drill-down to
evidence.

## 8.2 POC Requirements
The POC shall answer questions **scoped to provider-native data**, e.g. "which nodes are throttling right
now?", "show idle GPUs by tenant", "what's the blast radius of r14-n03?". Each answer links to the map/
evidence. It shall **not** attempt the V2 deep "why is my job slow" analysis (that needs the framework tier)
and shall say so when asked.

## 8.3 User Experience
**Screen:** `Ask` (a panel available across the app)

**Workflow:**
1. Operator types "which tenants have the most idle GPUs?".
2. The answer returns a ranked list with a link into the Utilization screen for each tenant.
3. "What did GPU-3 on r14-n03 do in the last hour?" returns the finding + a link to the evidence drawer.

## 8.4 Target Demonstration Outcome
Natural-language access to the same connected data — answers, not a query console — with the drill-down
intact.

---

# 9. Scenario 6: Alerting & On-call Routing

## 9.1 Demonstration Requirements
Show that a failure produces a **de-duplicated, blast-radius-aware** alert to the right channel — not noise.

## 9.2 POC Requirements
The POC shall:
- Generate alerts from findings with **blast-radius context** (which tenants/jobs).
- **De-duplicate** correlated findings (one root cause → one alert, not one per GPU).
- Route to **Slack / Teams / PagerDuty / webhook** (same channel set as Trillo Agent Observability), by
  configurable rule (severity, tenant, cluster).
- Let the operator open the alert straight into the Blast Radius / Reliability screens.

## 9.3 User Experience
**Screen:** `Alerts` + channel config

**Workflow:**
1. A rack-cooling event degrades 6 GPUs; the POC emits **one** alert ("rack r14 cooling — 6 GPUs throttling,
   3 tenants affected"), not six.
2. The Slack message carries the blast radius and a deep link back into the app.

## 9.4 Target Demonstration Outcome
The evaluator sees signal, not noise: correlated, blast-radius-aware alerts that route correctly and link
back to evidence.

---

# 10. Scenario 7: Operator Executive Dashboard

## 10.1 Demonstration Requirements
A single view of fleet health, real utilization/waste, reliability, and capacity headroom — the operator's
P&L and risk at a glance.

## 10.2 POC Requirements
The POC shall present: fleet health summary (healthy / degraded / failed, cordon recommendations);
**real utilization vs GPU_UTIL** and reclaimable capacity (with $ where a rate is configured); reliability
(failures caught, tenant-visible incidents avoided, MTBF/MTTR trend); occupancy and **capacity headroom**;
per-tenant occupancy and at-risk GPU-hours. All figures drill down to the underlying screens.

## 10.3 User Experience
**Screen:** `Executive Dashboard` (the operator's home)

**Workflow:**
1. Operator lands here: "Fleet 96% healthy · real MFU 34% (GPU_UTIL 89%) · 58 reclaimable GPUs · 2 cordons
   recommended · 0 tenant-visible incidents this week".
2. Clicking "58 reclaimable GPUs" drills into Utilization; clicking a cordon opens Reliability.

## 10.4 Target Demonstration Outcome
An executive-legible summary that is one click from the evidence behind every number.

---

# 11. POC Data and Storage

## 11.1 PostgreSQL first, then OliverDB
The POC is built **PostgreSQL-first** — the fast, practical way to populate, query, and demo synthetic fleet
telemetry and operational data. **OliverDB** is introduced behind a **config / env switch** once telemetry
volume warrants the columnar hot path; the integration is **reused from Trillo Agent Observability** (already
built in a separate track), so the switch is essentially configuration, not a rebuild. The POC Postgres model
is **not** the final production telemetry architecture — OliverDB carries fleet-wide cardinality in
production; Postgres retains metadata/policy.

## 11.2 Consolidated Logical and Telemetry Data Model
Combines (1) **fleet entities** (topology, tenant, job, allocation, finding, alert), (2) **infrastructure
telemetry** in OTLP-shaped rows, and (3) **analytical entities** for rollups, baselines, findings, and
recommendations.

### 11.2.1 Core Relationship Model
```text
Region
  └── AZ
        └── Rack  ── PowerDomain / CoolingDomain
              └── Node
                    ├── GPU ── HBM / NVLinkDomain / PCIeRoot
                    ├── NIC
                    └── (firmware / driver versions)

FabricSwitch (leaf/spine/core) ── FabricLink ──► Node / GPU        (IB / RoCE / NVLink)

Tenant (AOS system class, RLS) ── TenantProfile
     └── Job (K8s / Slurm) ── Allocation ──► Node / GPU            (blast-radius join)

OtlpTelemetry (span/log/event/metric)  ── executionId / jobRunId ──►  Job
     │  flattened fleet semconv: gpuUuid, nodeName, rackId, azId, regionId,
     │  fabricSwitchId, schedulerJobId, tenantId, dcgmField, …
     └──► MetricRollup / Baseline ──► Finding ──► Alert / AIAnalysis
```

Background intelligence relationship (functions own facts; agents own reasoning):
```text
SweeperRun / Function  →  Deterministic Rollup / Finding  →  Specialized AI Agent (when useful)  →  AIAnalysis / Recommendation
```

### 11.2.2 Telemetry & Correlation Modeling Rule
- Infra telemetry lands in a **single wide-column `OtlpTelemetry` table** (§11.3) — one row per
  span/log/event/metric point, signal-specific columns nullable — reusing the Agent Observability shape.
- Exporters do **not** share a trace id, so a first-class **`executionId` / `jobRunId`** column is the
  correlation spine. Ingest derivation, mirroring `OtlpTelemetry.executionId`:
  (a) promote `attrs.trillo.job_run_id` if the producer set it;
  (b) else derive `UUIDv5(trillo_ns, scheduler.namespace ‖ schedulerJobId)` when a scheduler job id is
  present;
  (c) else null (typical for fleet-wide metrics with no job context).
- The **scheduler `Allocation`** (job → node/GPU, with time bounds) is what lets a DCGM metric on a GPU be
  attributed to a job/tenant even though DCGM itself knows nothing about jobs — this is the blast-radius
  join, and it needs **no framework instrumentation**.
- `Finding` / `Alert` correlate back via the graph node id (`gpuUuid` / `nodeName` / `fabricLinkId`) plus
  `executionId` when a job is involved.

### 11.2.3 `region` / `az` / `rack`
Key fields: `id`, `name`, parent id, `powerDomain`, `coolingDomain`, `failureDomain`, `createdAt`,
`updatedAt`.

### 11.2.4 `node`
Key fields: `nodeName`, `rackId`, `cpuCount`, `ramGb`, `numaLayout`, `bmcAddress`, `nicIds`,
`firmwareVersion`, `driverVersion`, `status`, `firstSeen`, `lastSeen`.

### 11.2.5 `gpu`
Key fields: `gpuUuid`, `gpuIndex`, `nodeName`, `model`, `hbmGb`, `nvlinkDomainId`, `pcieRoot`, `health`,
`eccMode`, `rowRemapAvailable`, `status`, `firstSeen`, `lastSeen`.

### 11.2.6 `fabric_switch` / `fabric_link`
`fabric_switch`: `id`, `tier` (leaf/spine/core), `region/az`, `ports`, `status`.
`fabric_link`: `id`, `type` (ib/roce/nvlink), `endpointA`, `endpointB`, `rail`, `status`.

### 11.2.7 `tenant` / `tenant_profile`
`tenant` = AOS system class (`tenantId`, RLS). `tenant_profile`: `tenantId`, `type`
(external-customer/internal-team), `billingMode`, `slaTerms`, `contacts`.

### 11.2.8 `job` / `allocation`
`job`: `jobId`, `tenantId`, `scheduler` (k8s/slurm), `type` (training/inference), `status`, `startedAt`,
`endedAt`, `requestedGpus`, `executionId`.
`allocation`: `id`, `jobId`, `gpuUuid` / `nodeName`, `startedAt`, `endedAt`.

### 11.2.9 `finding` / `alert`
`finding`: `id`, `detector`, `severity`, `entityType`, `entityId`, `executionId`, `evidence` (JSON),
`recommendedAction`, `status`, `createdAt`.
`alert`: `id`, `findingIds[]` (dedup group), `blastRadius` (JSON), `channel`, `routedAt`, `status`.

## 11.3 Infrastructure Telemetry Schema (wide-column OTLP)
The POC reuses the Agent Observability **`OtlpTelemetry`** wide table — one table for `span / log / event /
metric` with a `signalType` discriminator and signal-specific nullable columns — because fleet telemetry
arrives as OTLP:
- **DCGM / NVML** exporter → OTLP **metrics** (GPU util, SM-active, tensor-active, temp, power, ECC counts,
  row-remap, throttle reasons, NVLink throughput).
- **Node exporter** → OTLP **metrics** (CPU, RAM, storage IOPS, PTP/NTP offset, NUMA).
- **Host / kernel** (Xid, dmesg) → OTLP **logs / events**.
- **Fabric Manager / switch** → OTLP **metrics / events** (NVSwitch errors, link status, basic counters).
- **Scheduler (K8s / Slurm)** → OTLP **events** (allocation, gang-schedule, preemption, job lifecycle).

**Fleet-specific flattened columns** are added to the wide table (the analog of the GenAI semconv columns in
Agent Observability), promoted from `resourceAttributes` / `attributes` at ingest for vectorized filtering:
`gpuUuid`, `gpuIndex`, `nodeName`, `rackId`, `azId`, `regionId`, `nvlinkDomainId`, `fabricSwitchId`,
`fabricLinkId`, `schedulerJobId`, `tenantId`, and `dcgmField` (the DCGM/metric field name, e.g.
`DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_PROF_SM_ACTIVE`, `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL`).
**`executionId`** is first-class as in §11.2.2. The generic OTLP columns (`traceId`, `spanId`, `timeUnixNano`,
`attributes`, metric value/bucket columns, etc.) are inherited unchanged from `OtlpTelemetry`.

## 11.4 Analytical, Governance, and Background Entities
- `metric_rollup` — per-entity/time-bucket aggregates (utilization, temp, error rates) for fast dashboards.
- `utilization_rollup` — real-utilization (MFU / SM-active / occupancy) and reclaimable-idle by fleet /
  cluster / tenant / time.
- `tenant_usage` — GPU-hours and occupancy by tenant (metering feed; **billing is V2**).
- `platform_finding` — deterministic detector output (see §11.2.9).
- `ai_analysis` — grounded reasoning attached to a finding (root-cause summary).
- `analysis_baseline` — per-entity "normal" for anomaly/degradation comparison.
- `capacity_estimate` — occupancy trend + headroom (light V1; forecast is V2).
- `alert_rule` — routing/dedup configuration.
- `sweeper_run` — background job execution state (topology reconcile, rollups, baselines).

### 11.4.x Recommended POC Tables / Trillo AOS Entities
**Fleet / inventory:** `region`, `az`, `rack`, `node`, `gpu`, `fabric_switch`, `fabric_link`, `tenant`,
`tenant_profile`, `job`, `allocation`.
**Telemetry:** `otlp_telemetry` (wide) — or the four per-signal tables if the engine prefers; the ingest
plugin fills the same flattened + `executionId` columns either way.
**Analytics / intelligence:** `metric_rollup`, `utilization_rollup`, `tenant_usage`, `platform_finding`,
`ai_analysis`, `analysis_baseline`, `capacity_estimate`, `sweeper_run`.
**Alerting / config:** `alert`, `alert_rule`.

## 11.5 Synthetic Dataset Requirements
The fleet simulator should make the product feel operationally real and keep one consistent story across all
seven scenarios. Recommended characteristics:
- A fleet of **several racks / hundreds–low-thousands of GPUs** across ≥1 region and multiple NVLink domains.
- A leaf-spine **IB/RoCE fabric** (switches + links) with realistic topology.
- **Multiple tenants** (external-customer and internal-team types), each with K8s inference and Slurm
  training jobs; a realistic `allocation` history.
- **≥30 days of history** for utilization, temperature, and error trends (to feed baselines and the exec
  dashboard).
- **Injected failure scenarios**, each mapped to a detector and a demo beat:
  - Rising ECC → row-remap exhaustion (Scenario 2 cordon-before-panic).
  - Xid 79 GPU-off-the-bus mid-training-run.
  - Silent-data-corruption suspect (DCGM anomaly on a GPU running a training job whose loss later spikes).
  - Rack cooling event → correlated multi-GPU thermal throttle (Scenario 6 dedup).
  - NCCL/straggler stall on a Slurm job.
  - Storage-starvation (VRAM-full / SM-idle) on an inference deployment (Scenario 4).
  - Fragmentation / gang-scheduling deadlock leaving GPUs idle.
- **The utilization illusion baked in:** high `GPU_UTIL`, low real MFU across a meaningful fraction of the
  fleet, with reclaimable idle concentrated in one or two tenants.

---

# 12. Trillo AOS Role in the POC
The POC is a **Trillo AOS application** — which is the point of the product's "easy to customize" wedge:
- **Entity model** (§11) is declarative metadata; data-service APIs, security, and deploy come from the
  runtime.
- **Multi-tenant + RLS** are native: `tenantId` isolation makes the V2 tenant edition a config change, not a
  fork.
- **Detectors, rollups, and correlation run as AOS functions** on pods / micro-VMs — the customization story
  (custom metrics/detectors/billing rules as functions, hot-deployed) is demonstrable here.
- **RBAC** governs the operator vs. (V2) tenant surfaces; actuation (V2) is a gated, high-`effect`,
  `sensitive` capability.
- **Deploy** is via the AOS deploy flow; **scoped chat** uses the AOS-native AI surface / MCP.
- **In your environment (BYOC):** collectors + Postgres + OliverDB run in the provider's own cloud; fleet and
  tenant data never leave it.

---

# 13. POC Background Intelligence

## 13.1 Execution Modes
Detectors and rollups run as **scheduled** (sweeps: reconcile topology, roll up utilization, refresh
baselines) and **event-driven** (a threshold breach or a scheduler event creates a finding immediately)
functions.

## 13.2 Deterministic Detection, Agentic Investigation
- **Functions own facts:** the detector catalog (§5.2) is deterministic — thresholds, trends, and joins over
  telemetry produce findings with evidence.
- **Agents own interpretation:** a specialized AI analysis may attach a grounded root-cause summary to a
  finding, and the scoped chat may call trusted functions as tools to fetch evidence. Reasoning never invents
  facts.

## 13.3 Background Services
- **Topology reconcile sweeper** — telemetry-discovered graph vs. authoritative sources (K8s + Slurm APIs;
  labels/CMDB for physical); fabric authoritative source is V2.
- **Utilization rollup** — real-utilization + reclaimable idle by fleet/cluster/tenant.
- **Baseline refresh** — per-entity "normal" for degradation/anomaly comparison.
- **Blast-radius resolver** — device → allocation → job → tenant on demand and on failure.

## 13.4 Findings Rather Than Alerts
The POC produces **findings** (durable, evidence-backed, deduplicated) as the primary object; **alerts** are
routed views of findings. This avoids alert-fatigue: one root cause → one finding → one alert, with the blast
radius attached.

## 13.5 Recommendation, Not Actuation
Findings carry a **recommended action** ("cordon r14-n03", "reclaim 40 idle GPUs from tenant Acme"). V1
stops there — the operator acts. Actuation is the V2 trust ladder.

## 13.6 Scheduling and Batching
Sweeps run on the AOS scheduler (multi-app cron); event-driven detectors fire on ingest. Rollups are batched
to keep the exec dashboard responsive on synthetic data.

---

# 14. Demonstration Storyline

1. **Start on the Executive Dashboard** — "Fleet 96% healthy · real MFU 34% vs GPU_UTIL 89% · 58 reclaimable
   GPUs · 2 cordons recommended · 0 tenant-visible incidents this week."
2. **Investigate a failure** — drill into a cordon recommendation; rising ECC → row-remap exhaustion, with
   evidence. Cordon *before* the panic.
3. **Blast radius** — one click: which jobs and tenants this GPU threatens, across K8s and Slurm.
4. **Utilization** — the illusion: GPU_UTIL 91% / MFU 27%; 40 reclaimable GPUs concentrated in one tenant.
5. **Ask the Fleet** — "which tenants have the most idle GPUs?" — natural-language, grounded, drillable.
6. **Alerting** — a rack-cooling event fires *one* blast-radius-aware alert to Slack, not six.
7. **Close on the Fleet Map** — the whole fleet, connected, healthy again after the recommended cordon.

---

# 15. POC Success Criteria

The POC is a success when, in the provider's own environment (or on synthetic data for the demo), Trillo can:

1. **Auto-discover** the fleet and render the live topology map (compute + physical + NVLink + fabric),
   with tenant/job overlay from **both K8s and Slurm**.
2. **Detect** the V1 provider-native failure set and **recommend cordon before a panic** on at least the
   ECC/row-remap and thermal-throttle paths — with evidence.
3. **Attribute blast radius** — the exact tenant(s)/job(s) a failing device affects — with **no framework
   instrumentation**, RLS-scoped.
4. **Surface real utilization** — MFU vs `GPU_UTIL` — and quantify reclaimable idle per tenant and cluster.
5. **Alert** with de-duplicated, blast-radius-aware routing to the configured channels.
6. **Answer** fleet questions via scoped chat, with drill-down to evidence.
7. Run entirely as a **Trillo AOS application**, **Postgres-first with the OliverDB switch available**, on
   **synthetic fleet telemetry** driven by the simulator (§11.5).
