# Trillo Neoclouds Observability — Authoring-Heavy Function Specs

**Status:** reference specs for the three functions whose logic is subtle enough to pin down before the team
implements/regenerates them (the physical-topology builder, the logical-topology projection, and the
LCA blast radius). Grounded in the `aos_toolkit` conventions seen in the generated functions
(`ctx.data.query(cls, filters=[{field,op,value}], start, end)`, `ctx.data.create(cls, dict)`,
`ctx.data.get(cls, id)`, `ctx.data.update(cls, patch)`, `ctx.success/error/not_found`, `ctx.now()`).
Companion to `...-Plan.md` §6/§7. App: `NeoCloudObservability` `.trillo/577`.

---

## 0. Function-layer impact of the Slice 0 rename (do this first)
The Slice 0 schema change (`tenantProfileId` → `tenantId`/`refTenantId`; `aosTenantId` → `tenantId` on the
`TenantProfile` sidecar; `targetTenantProfileId` → `targetTenantId`; new `TopoNode`/`TopoEdge`/
`AllocationTopoMember`) touches **16 existing functions** that reference `tenantProfileId`. **Regenerate the
function + UI layer via the Claude Code plugin** from the updated entities + this spec rather than
hand-patching. Field mapping for the regeneration:
- **Tenant-scoped classes** (`Job`, `Allocation`, `TenantUsage`): `tenantProfileId` → **`tenantId`** (the RLS
  key / framework `Tenant` id).
- **Operator-global classes** (`PlatformFinding`, `Alert`, `AlertRule`, `UtilizationRollup`, `MetricRollup`,
  `OtlpTelemetry`, `AllocationTopoMember`, `SimulationScenario`): → **`refTenantId`** / `targetTenantId`
  (denormalized hint, not RLS).
- **Getting a tenant's profile:** query `TenantProfile` by **`tenantId`** (the sidecar key), not by its own
  `id`. (`select * from TenantProfile where tenantId = :tid`.)
- `resolve_blast_radius` is not just renamed — it is **rewritten** per §3 below.

---

## 1. `build_physical_topology` — inventory → the one physical graph (Slice 1/2)
**Purpose:** create one `TopoNode` per inventory entity + `TopoEdge`s for containment and connectivity.
**Idempotent** (upsert by `sourceType`+`sourceRef`), so it can re-run after inventory changes.

**Vertices** — for each row, create/keep a `TopoNode`:
- `Region`, `AvailabilityZone`, `Rack`, `Node` (sourceType `compute_node`), `Gpu`, `FabricSwitch`,
  `StorageSystem` → `{sourceType, sourceId=row.id, sourceRef=<natural key>, name, rackId, regionId,
  status, validFrom=now}`. Natural keys: `Gpu.gpuUuid`, `Node.nodeName`, switch/storage name/slug, rack/az/
  region id.
- **NVLink domains**: `distinct Gpu.nvlinkDomainId` → `TopoNode{sourceType=nvlink_domain, sourceId=null,
  sourceRef=<domainId>, rackId=<the domain's rack>}` (domain is intra-rack — assert it).

**Edges** — create/keep `TopoEdge`s:
- **Containment** (`edgeType=contains`, `directed=true`): region→az→rack→node→gpu; rack/az→storage where placed.
- **NVLink** (`edgeType=nvlink`, undirected): each `Gpu` ↔ its `nvlink_domain` vertex.
- **Compute fabric** (`edgeType=compute_fabric`, undirected): from `FabricLink` — node-vertex ↔ switch-vertex,
  carrying `linkId=link.id`, `rail=link.rail`; and leaf↔spine switch ↔ switch links.
- **Storage fabric** (`edgeType=storage_fabric`): each node ↔ the storage systems it mounts (demo: connect
  each node to the region's `shared=true` storage).

**Idempotency:** before create, `ctx.data.query("TopoNode", filters=[sourceType eq, sourceRef eq], end=1)`;
same shape for edges by `(fromNodeId, toNodeId, edgeType)`. Return `{nodes:n, edges:m}`.

---

## 2. `resolve_logical_topology` — allocation → `AllocationTopoMember` (Slice 4)
**Purpose:** for a job-run, materialize its logical topology = **endpoint** vertices (held GPUs/nodes) + the
**traversed** shared vertices (the components its traffic crosses). Call on scheduler-event ingest, or
on-demand for a job.

**Inputs:** `jobId` or `executionId`, optional `asOf` (default now).

**Logic:**
1. Load the job's **active allocations** at `asOf`:
   `ctx.data.query("Allocation", filters=[{jobId eq} OR {executionId eq}, {startedAt le asOf},
   {endedAt is_null} OR {endedAt gt asOf}])`. Read `job.tenantId`.
2. **Endpoint members** — for each allocation: emit `AllocationTopoMember{jobId, executionId,
   refTenantId=job.tenantId, allocationId=a.id, topoNode=<gpu vertex>, topoSourceType=gpu,
   topoSourceRef=a.gpuUuid, role=endpoint, validFrom=a.startedAt, validTo=a.endedAt}`; also the **node**
   vertex (role=endpoint) and the gpu's **nvlink_domain** vertex (role=traversed).
3. **Traversed members** — project onto the graph:
   - Collect the distinct **nodes** the job spans → their **leaf switches** via `compute_fabric` `TopoEdge`s
     → role=traversed.
   - If the job spans **>1 rack/leaf**, add the **spine** switches on the leaf-to-leaf paths (demo: the
     spine tier serving those leaves) → role=traversed.
   - Add **storage systems** the job's nodes mount (`storage_fabric` edges) → role=traversed.
4. Upsert `AllocationTopoMember` by `(jobId, topoNodeId, role)`. Return `{members:k, endpoints, traversed}`.

This is the "logical topology created from allocation." It is derived and cheap to recompute; it is *not* a
stored subgraph.

---

## 3. `resolve_blast_radius` — LCA over the graph + membership (Slice 4, rewrite)
**Purpose:** given a failed `TopoNode` (or finding/device), find affected job-runs + tenants via
`AllocationTopoMember`, and for multiple failures compute the **lowest-common-denominator root cause**.

**Inputs:** `findingId` **or** `topoSourceType`+`topoSourceRef` (or legacy `entityType`+`entityId`, mapped to
a vertex); optional list `findingIds` for multi-failure LCA; `asOf` (default now).

**Single-failure logic:**
1. Resolve the failed `TopoNode`. Compute its **affected vertex set** by traversing `TopoEdge`s *outward*:
   - `contains` edges: descendants (rack→nodes→gpus).
   - `compute_fabric`/`nvlink`/`storage_fabric` edges: the vertices whose traffic depends on it (a leaf
     switch → the nodes/gpus it serves; an nvlink_domain → its gpus). Direction: follow edges to the
     "served" side; a switch failure affects the nodes on its links.
2. For each affected vertex, look up **`AllocationTopoMember`** active at `asOf`
   (`filters=[{topoNodeId in ...}, {validFrom le asOf}, {validTo is_null OR gt asOf}]`).
3. Aggregate per **job-run** (`jobId`/`executionId`): impact = **`direct`** if any hit member is
   `role=endpoint`; **`degraded`/`potential`** if only `role=traversed`. Tenant from `refTenantId`
   (resolve name via `TenantProfile` where `tenantId=refTenantId`). GPU-hours at risk = remaining × gpuCount.
4. Roll up tenants; pick primary tenant by hours. Persist `blastRadius` JSON + `refTenantId` on the finding.

**Multi-failure / root-cause (LCA):** given `findingIds` (or several correlated failures):
1. For each failure, get its affected job-run set (above).
2. Root cause = the **minimal set of shared `TopoNode`s that covers all affected job-runs** — i.e., walk each
   affected job-run's `AllocationTopoMember` (its logical topology), intersect, and take the **lowest common
   ancestor(s)** in the graph (the smallest/most-specific vertex — a leaf switch beats "the whole rack" beats
   "the region"). Score candidates by (covers-all, most-specific, `role=traversed` shared component).
3. Return `{rootCause: [{topoSourceType, topoSourceRef, covers: [jobIds]}], jobs, summary, highlight}`.

**Return shape** (superset of the current function): `{entity, summary{gpuCount, jobCount, tenantCount,
tenants, gpuHoursAtRisk, primaryRefTenantId}, jobs[], rootCause[], highlight{topoNodes, edges}, computedAt}`.

---

## 4. Notes
- Everything above is **operator-global** reads (the operator sees all tenants); tenant views (Slice 9)
  reuse the same functions filtered to the tenant's `AllocationTopoMember` rows.
- For demo scale (hundreds of GPUs, tens of jobs) the graph traversals are cheap; the `AllocationTopoMember`
  index keeps failure→jobs an indexed lookup rather than a full-graph walk.
- Keep `build_physical_topology` and `resolve_logical_topology` runnable from the simulator (Slice 1) so the
  synthetic fleet lands the graph + memberships through the same path real ingest will use.
