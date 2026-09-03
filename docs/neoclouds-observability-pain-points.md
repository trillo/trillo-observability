# Neoclouds & Private AI Cloud Observability — Exhaustive Pain-Point Catalog

**Purpose:** the full, unbounded superset of pains we see across GPU fleets and the teams that run or rent
them. Source material for posts/carousels, the PRD, and the brochure — the LinkedIn "9 pain points" post is
a curated subset of this. Organized by layer and concern; spans **both personas** — the **operator** (runs
the fleet) and the **tenant / customer** (runs jobs on it). Not everything here is V1; see *Detectability* at
the end for how these map to what's observable provider-native vs. what needs tenant/framework
instrumentation.

**Companion docs:** `neocloud_private_ai_cloud_glossary.md` (components + efficiency/telemetry vocab),
`Trillo-Neoclouds-Observability-PRD.md` (V1/V2 scope), the AI-infra root-cause matrix (25 tagged failure
modes), `Trillo-Neoclouds-Observability-Brochure.md`.

---

## 1. GPU & silicon health
- **Double-bit / uncorrectable ECC errors** panic the host and kill the job outright.
- **HBM row-remap exhaustion** — spare rows run out; the GPU is quietly at end-of-life.
- **GPU falls off the bus (Xid 79)** mid-run, freezing or crashing the job.
- **Silent Data Corruption (SDC)** — a marginal GPU does bad math with no error; loss goes NaN hours later.
- **NVSwitch / Fabric-Manager faults** drop intra-node GPU-to-GPU comms to zero.
- **Firmware / driver drift** across the fleet — inconsistent versions cause subtle instability.
- **PCIe link degradation** (downtrained lanes) silently halves a GPU's bandwidth.
- **Single-bit ECC creep** — rising correctable errors foreshadow a failure nobody is trending.
- **Repeat-offender nodes** fail again and again because no one has quarantined them.

## 2. Memory (HBM / VRAM)
- **VRAM leaks from zombie processes** — a dead job's memory is never released, blocking scheduling.
- **OOM kills that look random** because there's no per-process VRAM history.
- **VRAM fragmentation** blocks large allocations despite "free" memory.
- **KV-cache exhaustion** (inference) silently drops throughput or rejects requests.

## 3. Utilization & efficiency (the "utilization lie")
- **GPU_UTIL says busy; real MFU is a fraction** — wasted compute you can't see.
- **Low SM occupancy** while the dashboard reads ~100%.
- **Tensor cores idle** while CUDA cores spin — kernel/precision inefficiency invisible.
- **Allocated-but-idle GPUs** — reserved, not running, and nobody reclaims them.
- **Memory-bound / low-arithmetic-intensity kernels** leave the GPU underfed.
- **No fleet-wide "real utilization" number** anyone actually trusts.
- **HFU vs MFU gap** — redundant work (recompute) inflating apparent efficiency.

## 4. Distributed training & collective communication
- **NCCL / RCCL ring deadlocks** — hundreds of GPUs idle waiting on one peer.
- **Stragglers** drag the whole synchronous job to the slowest link.
- **Gradient-sync stalls** from network jitter you can't localize.
- **No "patient zero"** — can't identify which node hung the collective.
- **Elastic / failed-worker churn** repeatedly restarts costly training from the last checkpoint.
- **Poor overlap of compute and communication** silently caps scaling efficiency.

## 5. Network & fabric
- **The fabric is a black box** — GPU metrics, but no InfiniBand / RoCE visibility.
- **Lossless-network packet drops / PFC storms** silently inflate tail latency.
- **Asymmetric routing / rail imbalance** — identical nodes run at different speeds.
- **Congestion / incast** on all-reduce overruns switch and NIC buffers.
- **Oversubscription you can't measure** — bisection bandwidth quietly bottlenecks big jobs.
- **NUMA / NIC-GPU misalignment** forces cross-socket hops and latency spikes.
- **PTP / NTP clock drift** corrupts distributed traces and triggers spurious timeouts.
- **No topology-aware view** linking a job to the switches and links it actually traverses.

## 6. Storage & data pipeline
- **Storage stalls that look like GPU problems** — GPUs wait on data; you blame the model.
- **Shared-filesystem contention** (Lustre / WEKA / VAST) — a noisy neighbor starves everyone.
- **Checkpoint I/O stalls** — multi-TB writes periodically freeze the job.
- **Object-store / cache latency** throttles data loaders.
- **GPUDirect Storage misconfigured** — data routes through the CPU, wasting bandwidth.
- **No per-job storage throughput** measured against the GPU feed rate.

## 7. Power, thermal & cooling / facility
- **Thermal throttling you can't see** downclocks a node with no alarm ("healthy but slow").
- **Power capping (TGP)** limits performance before any hard alert fires.
- **Rack / PDU power-headroom unknown** — you oversubscribe power blind.
- **Cooling faults (CDU / DLC)** raise temps across a rack — correlated failures.
- **No facility correlation** — a power/cooling event looks like scattered GPU issues.
- **Energy cost per job / per tenant is invisible** — a growing FinOps and sustainability concern.

## 8. Scheduling, queueing & multi-tenancy
- **Fragmentation** — free GPUs exist but scattered; tightly-coupled jobs can't schedule.
- **Gang-scheduling deadlocks** — partial allocations held while waiting for the rest.
- **Long queue waits / slow time-to-first-batch** with no visibility into why.
- **Preemption loops** thrash low-priority jobs.
- **Quota / fair-share disputes** with no per-tenant accounting to settle them.
- **Topology-unaware placement** scatters a coupled job across slow links.
- **Noisy neighbors** — one tenant's job degrades another's on shared hardware.

## 9. Reliability, failure & recovery
- **Failures your customers / tenants find before you do.**
- **No blast radius** — can't say which jobs or tenants a failed device or link just hit.
- **Slow root cause** — hours spent correlating hardware, job, and fabric by hand.
- **No burn-in / qualification signal** — repaired hardware returns to prod unproven.
- **Checkpoint-restart pain** — recovery is slow, manual, or loses progress.
- **No automated drain / cordon** — a known-bad node keeps taking work.

## 10. Correlation & single source of truth (cross-layer)
- **No single source of truth** — GPU metrics here, network there, scheduler elsewhere.
- **Can't trace hardware → job → tenant** across the stack.
- **Metrics without topology** — numbers with no map to place them on.
- **Tool sprawl** — Prometheus + DCGM + vendor dashboards that don't talk to each other.
- **Can't answer "why is this job slow?"** across layers (GPU vs fabric vs storage vs contention).

## 11. Prediction & prevention
- **No early warning** — you catch failures after they hit, not before.
- **Degradation trends untracked** (ECC creep, thermal drift, remap consumption) → no prediction.
- **No RUL / predictive maintenance** — you replace on failure, not on forecast.
- **No per-device baseline of "normal"** to detect anomalies against.

## 12. Cost, billing, margin & capacity
- **Utilization you can't bill or defend** — GPU-hours unaccounted.
- **Margin leaking** from idle / underused capacity you can't see.
- **Chargeback / showback by spreadsheet** — internal cost allocation is guesswork.
- **Capacity planning by vibes** — buy more vs. pack tighter, with no data.
- **Oversubscription blind spots** — safe headroom unknown.
- **No per-tenant cost / margin view** to inform pricing.

## 13. SLA & tenant experience
- **Can't measure or report per-tenant reliability / availability.**
- **SLA breaches discovered from complaints**, not telemetry.
- **No tenant-facing visibility** — customers fly blind on their own jobs and hardware.
- **Noisy-neighbor SLA erosion** invisible to both tenant and operator.

## 14. Security, isolation & compliance
- **No audit trail** of who/what accessed which GPUs and data.
- **Tenant isolation you can't verify** (network, storage, memory).
- **Data residency** — telemetry / tenant data leaving the environment.
- **Compliance evidence** (access, retention, encryption) hard to produce on demand.

## 15. Fleet lifecycle & operations
- **Bring-up / provisioning** — new nodes fail qualification silently.
- **Firmware / driver drift management** is manual and error-prone.
- **Drain / quarantine / repair workflows** are ad-hoc and untracked.
- **Inventory & topology discovery** is stale or hand-maintained.
- **Heterogeneous fleet** (GPU generations, rented + private) with no unified view.

## 16. Inference / serving-specific
- **Continuous-batching stalls** drop goodput with no alert.
- **KV-cache exhaustion / eviction thrash** under load.
- **Latency-SLO misses (p99)** with no attribution to hardware vs. model vs. contention.
- **Autoscaling lag** — cold starts and capacity mismatches under bursty demand.
- **Memory-bound decode** leaves compute idle (low MBU).

## 17. Tooling fit & operability (the meta-pain)
- **Rigid tools don't fit your fleet;** DIY observability becomes a second full-time job.
- **Building your own** = a data-pipeline project you never wanted to own.
- **Custom metrics / detectors / billing rules** are impossible to add without forking.
- **Alert fatigue** — noise without blast-radius-aware dedup and routing.

---

## Detectability — how these map to observation

Two tiers, which is also the V1/V2 seam in the PRD:

- **Provider-native (deployable fleet-wide, zero tenant cooperation):** everything sourced from
  **DCGM/NVML, node exporters, host/kernel logs, the scheduler (K8s + Slurm), Fabric Manager, and switch
  telemetry**. Covers most of §1, §2, §3, §5, §7, §8, §9, §10, §12–§15. This is the V1 surface.
- **Tenant / framework-instrumented (needs the workload owner's buy-in):** anything requiring
  **Ray / PyTorch / vLLM** signal — full **SDC confirmation via loss/gradient correlation** (§1), NCCL
  "patient-zero" (§4), the deep "why is my job slow" answer (§10), and most serving-internal metrics (§16).
  This is the V2 tenant tier.

Facility (§7) deep integration (BMS) is V2; V1 **infers** facility issues from correlated multi-node thermal
anomalies.

## Personas — who feels each pain
- **Tenant / workload team** (engineer): §1–§6, §16 — "is my job healthy, and is their hardware hurting me?"
- **Operator / platform / SRE:** §7–§11, §15 — fleet health, blast radius, capacity, ops.
- **Manager / FinOps / exec:** §12–§14, §17 — margin, billing, SLA, buy-vs-build.

## Curation notes (for posts / carousels)
- The **9-pain LinkedIn post** draws its punchiest, most-differentiated items from §3 (utilization lie),
  §1 (SDC, bad GPUs), §4 (stragglers), §6 (storage), §9 (blast radius), §10 (source of truth), §12
  (billing, capacity).
- Sharpest, most-shareable hooks: **the utilization lie (§3)** and **Silent Data Corruption (§1)**.
- For a follow-up post, **§5 (fabric black box)** and **§6 (storage stalls)** are the strongest untold half
  of the pipeline story.
