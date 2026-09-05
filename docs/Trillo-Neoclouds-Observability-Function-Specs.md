# Trillo Neoclouds Observability — Authoring-Heavy Function Specs

**Status:** reference specs for the three functions whose logic is subtle enough to pin down before the team
implements/regenerates them (the relationship builder, the logical-topology projection, and the LCA blast
radius). Grounded in the `aos_toolkit` conventions seen in the generated functions
(`ctx.data.query(cls, filters=[{field,op,value}], start, end)`, `ctx.data.create(cls, dict)`,
`ctx.data.get(cls, id)`, `ctx.data.update(cls, patch)`, `ctx.success/error/not_found`, `ctx.now()`).
Companion to `...-Plan.md` §6/§7. App: `NeoCloudObservability` `.trillo/577`.

**Model recap.** Topology is a set of typed **relationships between components**, not a connectivity graph.
`ComponentRelation` relates two typed components directly — `(fromType, fromRef)` ↔ `(toType, toRef)` with a
`kind` (`contains` / `nvlink` / `compute_fabric` / `storage_fabric` / …). The typed entities (`Gpu`, `Node`,
`FabricSwitch`, `StorageSystem`, …) are the components; refs are their natural keys (`gpuUuid`, `nodeName`,
switch/storage name, rack id). "What is X related to?" = query `ComponentRelation` on either endpoint.

---

## 0. Function-layer impact of the Slice 0 rename (do this first)
The Slice 0 change (`tenantProfileId` → `tenantId`/`refTenantId`; `aosTenantId` → `tenantId` on the
`TenantProfile` sidecar; `targetTenantProfileId` → `targetTenantId`; new `ComponentRelation` /
`AllocationMember` / `Report` / `StorageSystem`) touches **16 existing functions** that reference
`tenantProfileId`. **Regenerate the function + UI layer via the plugin** from the updated entities + this
spec rather than hand-patching. Mapping:
- **Tenant-scoped classes** (`Job`, `Allocation`, `TenantUsage`): `tenantProfileId` → **`tenantId`**.
- **Operator-global classes** (`PlatformFinding`, `Alert`, `AlertRule`, `UtilizationRollup`, `MetricRollup`,
  `OtlpTelemetry`, `AllocationMember`, `SimulationScenario`): → **`refTenantId`** / `targetTenantId`.
- **Getting a tenant's profile:** query `TenantProfile` by **`tenantId`** (the sidecar key), not its own `id`.
- `resolve_blast_radius` is **rewritten** per §3.

---

## 1. `build_physical_topology` — inventory → the component relationships (Slice 1/2)
**Purpose:** from inventory, create the `ComponentRelation` rows that relate components. **Idempotent**
(upsert by `fromType,fromRef,toType,toRef,kind`), so it re-runs after inventory changes.

Read inventory (`Region`, `AvailabilityZone`, `Rack`, `Node`, `Gpu`, `FabricSwitch`, `StorageSystem`; NVLink
domains = distinct `Gpu.nvlinkDomainId`). Component refs are their natural keys. Then create relations:
- **Containment** (`kind=contains`, `directed=true`): region→az, az→rack, rack→node, node→gpu; rack/az→storage.
- **NVLink** (`kind=nvlink`): each `Gpu` ↔ its `nvlink_domain`.
- **Compute fabric** (`kind=compute_fabric`): from `FabricLink` — node ↔ switch (carry `linkId`, `rail`); and
  leaf ↔ spine switch ↔ switch.
- **Storage fabric** (`kind=storage_fabric`): each node ↔ the storage it mounts (demo: node ↔ region's
  `shared=true` storage).

Idempotency: before create, `ctx.data.query("ComponentRelation", filters=[fromType eq, fromRef eq, toType eq,
toRef eq, kind eq], end=1)`. Return `{relations:n}`. **Neighbor query (UI):** "what is X related to" =
`ComponentRelation where (fromType=T and fromRef=R) or (toType=T and toRef=R)` → the related `(type, ref)`
pairs → resolve typed entity → icon.

---

## 2. `resolve_logical_topology` — allocation → `AllocationMember` (Slice 4)
**Purpose:** for a job-run, materialize its logical topology = the **endpoint** components (held GPUs/nodes)
plus the **traversed** shared components its traffic crosses. Call on scheduler-event ingest, or on-demand.

**Inputs:** `jobId` or `executionId`, optional `asOf` (default now). Logic:
1. Load the job's **active allocations** at `asOf` (`Allocation` where jobId/executionId matches, `startedAt
   le asOf`, `endedAt is_null or gt asOf`). Read `job.tenantId`.
2. **Endpoint members** — per allocation, emit `AllocationMember{jobId, executionId, refTenantId=job.tenantId,
   allocationId, componentType=gpu, componentRef=a.gpuUuid, role=endpoint, validFrom=a.startedAt,
   validTo=a.endedAt}`; also the **node** (role=endpoint) and the gpu's **nvlink_domain** (role=traversed).
3. **Traversed members** — walk `ComponentRelation` from the job's nodes: their **leaf switches**
   (`compute_fabric`), the **spine** switches if the job spans >1 rack, and the **storage** it mounts
   (`storage_fabric`) → `role=traversed`.
4. Upsert `AllocationMember` by `(jobId, componentType, componentRef, role)`. Return
   `{members, endpoints, traversed}`.

Logical topology is derived and cheap to recompute — not a stored subgraph.

---

## 3. `resolve_blast_radius` — LCA over relations + membership (Slice 4, rewrite)
**Purpose:** given a failed component (or finding/device), find affected job-runs + tenants via
`AllocationMember`, and for multiple failures compute the **lowest-common-denominator root cause**.

**Inputs:** `findingId` **or** `componentType`+`componentRef` (map legacy `entityType`+`entityId`); optional
`findingIds` for multi-failure LCA; `asOf` (default now).

**Single failure:**
1. Resolve the failed component. Compute its **affected component set** by walking `ComponentRelation`
   *outward*: `contains` relations → descendants (rack→nodes→gpus); `compute_fabric`/`nvlink`/`storage_fabric`
   relations → the components that depend on it (a leaf switch → the nodes/gpus on its links; an nvlink_domain
   → its gpus).
2. For each affected component, look up **`AllocationMember`** active at `asOf`
   (`componentType`+`componentRef` in the affected set, `validFrom le asOf`, `validTo is_null or gt asOf`).
3. Aggregate per **job-run**: impact = **`direct`** if any hit member is `role=endpoint`; **`degraded`/
   `potential`** if only `role=traversed`. Tenant from `refTenantId` (name via `TenantProfile` where
   `tenantId=refTenantId`). GPU-hours at risk = remaining × gpuCount. Persist `blastRadius` + `refTenantId`
   on the finding.

**Multi-failure / root cause (LCA):** given several failures / `findingIds`:
1. Get each failure's affected job-run set (above).
2. Root cause = the **minimal set of shared components covering all affected job-runs** — walk each affected
   job-run's `AllocationMember` (its logical topology), intersect, and take the **lowest common denominator**
   (the smallest / most-specific shared component: a leaf switch beats "the rack" beats "the region"). Score by
   (covers-all, most-specific, `role=traversed`).
3. Return `{rootCause:[{componentType, componentRef, covers:[jobIds]}], jobs, summary, highlight}`.

**Return shape:** `{entity, summary{gpuCount, jobCount, tenantCount, tenants, gpuHoursAtRisk,
primaryRefTenantId}, jobs[], rootCause[], highlight{components, relations}, computedAt}`.

---

## 4. Notes
- All reads above are **operator-global** (the operator sees all tenants); tenant views (Slice 9) reuse the
  same functions filtered to the tenant's `AllocationMember` rows.
- For demo scale (hundreds of GPUs, tens of jobs) the relation walks are cheap; `AllocationMember` keeps
  failure→jobs an indexed lookup rather than a full walk.
- Run `build_physical_topology` and `resolve_logical_topology` from the simulator (Slice 1) so the synthetic
  fleet lands the relations + memberships through the same path real ingest will use.
