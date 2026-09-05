# Neoclouds Observability — Change Summary (team brief)

**Purpose:** bring the backend + UI teams up to speed on what changed since the app was first generated.
Deltas only — full detail is in the PRD, POC-Requirements, Plan, and Function-Specs docs. App:
`NeoCloudObservability` `.trillo/577` (branch `development/1.0`).

## Scope reframe
- **Demo build scope is unified** — the demo shows the full vision on synthetic data. The old V1/V2 split is
  now just a **deployment-readiness tier** (day-one / connector / maturity), not a build gate.
- **App is now multi-tenant** — `AppConfig.multiTenant` will be flipped **before seeding data**.

## Tenancy (important — affects most classes)
- **`TenantProfile` is now a 1:1 sidecar of the framework `Tenant`, keyed by `tenantId`.** `aosTenantId`
  retired.
- **App classes carry `tenantId`, never `tenantProfileId`.** Resolve profile attrs by joining `TenantProfile`
  on `tenantId`.
- **Tenant-scoped (RLS) classes:** `TenantProfile`, `Job`, `Allocation`, `TenantUsage` → **`tenantId`**.
- **Operator-global classes** (fleet, telemetry, rollups, findings, alerts) stay global and use a
  denormalized **`refTenantId`** hint (NOT `tenantId`, so RLS doesn't hide fleet-wide rows from the operator).
- Operator = **tenant-0** (sees all). Tenant views (later slice) = **"view as tenant"** impersonation with
  real RLS behind it.

## Topology (new model)
- Topology is a set of typed **relationships between components**: **`ComponentRelation`** relates two
  components directly — `(fromType,fromRef)`↔`(toType,toRef)` with a `kind`. Select a component → its
  relations → related components. (No vertex/edge graph; `TopoNode`/`TopoEdge`/`NodeToLink` retired.)
- Typed entities (`Gpu`, `Node`, `FabricSwitch`, `StorageSystem`, …) remain the source of truth; the graph is
  a thin index over them for traversal.
- **One physical topology** (built from inventory) + **many logical topologies** (per job-run, derived),
  materialized as **`AllocationMember`** (job-run ↔ component, `role` endpoint/traversed).
- **Blast radius / root cause = LCA** over the graph + membership (a shared switch/rack/storage failure fans
  out to every tenant it serves; multiple failed jobs resolve to one common root).

## New / changed entities
- **Add:** `ComponentRelation`, `AllocationMember`, `Report`, `StorageSystem`.
- **Changed:** `OtlpTelemetry` (+`storage` source, +`storageSystemId`, tenant hint → `refTenantId`);
  `SimulationScenario` (+`shared_fs_contention`, `targetTenantId`); the tenancy renames above.

## Backend / functions
- The rename touches **16 functions** that referenced `tenantProfileId`. **Regenerate the function layer via
  the plugin** (don't hand-patch). Mapping: tenant-scoped → `tenantId`, operator-global → `refTenantId`.
- Three functions have **detailed specs** (algorithms) in the Function-Specs doc: `build_physical_topology`,
  `resolve_logical_topology`, and the **LCA** `resolve_blast_radius`.

## For the UI team
- **UI will regenerate — expected and fine.** Overview of what's new so the regenerated screens make sense:
  - **Fleet Map** + a **Relationships** panel: select a component → what it's related to (via `ComponentRelation`), with icons.
  - **Blast Radius** now shows **direct vs shared** impact and a **root-cause (LCA)** component; opens from a
    finding or a map node.
  - **Allocation Timeline** ("who held what, when") is a first-class view (as-of-T).
  - **Utilization** leads with **real MFU vs `GPU_UTIL`** and reclaimable idle by tenant.
  - Tenant-facing views (later) are the same screens filtered by the tenant's `AllocationMember`.
  - Bindings use **`tenantId`** on tenant-scoped data and **`refTenantId`** on operator-global data.

## Full docs (in `trillo-observability/docs/`)
`...-PRD.md` (scope) · `...-POC-Requirements.md` (scenarios/acceptance) · `...-Plan.md` (build slices) ·
`...-Function-Specs.md` (the three subtle functions).
