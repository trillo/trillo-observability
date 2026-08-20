# TAO SRE Copilot — Tool Manifest

The curated set of MCP tools the **TAO SRE Copilot** exposes, and the mapping to
the real Trillo Agent Observability app functions. Source of truth for both the
AOS runtime MCP allow-list and the runbook skills.

Design reference: `./Enterprise_AI_Agent_Observability_SRE_Copilot_Plugin_Design.md`.

## Gating

- **App gate:** the app's `appName` must be on the catalog allow-list.
- **Item gate:** only functions with **`invocationMode == agent_tool`** are
  exposed. (No separate `sreExposable` flag — the metadata already carries it.)
- **RBAC:** each function declares `role` + `allowedAppRoles`; the OAuth'd AOS
  session enforces it, so masking is correct at the MCP boundary.
- **Redaction posture:** **structure, not payload, by default.** Tools return
  findings / clusters / spread / topology / span skeletons / rates / counts /
  baselines — never raw `prompt_text` / `completion_text` / tool payloads. Raw
  payload access is a separate, RBAC-gated, audited tool (not in v1).

## Read tools (from `invocationMode: agent_tool`)

| MCP tool | App function | Params | Returns (structure) |
| :-- | :-- | :-- | :-- |
| `get_top_findings` | `get_top_platform_findings` | filters (type/severity/status/time) | ranked PlatformFindings |
| `get_failure_cluster_stats` | `get_failure_cluster_statistics` | `{findingId}` | blast-radius: counts by status/agent/app + time span |
| `get_impacted_agents` | `get_impacted_agent_findings` | `{findingId}` | agents sharing the failing dependency |
| `get_execution` | `get_execution_details` | `{executionId}` | execution + span skeleton (category/timing/status) |
| `get_correlated_logs` | `get_correlated_logs_and_events` | `{traceId / executionId}` | correlated logs + events |
| `get_dependency_topology` | `get_dependency_topology` | scope | agent→model/tool/system topology |
| `get_agent_dependency_tree` | `get_agent_dependency_tree` | `{agentId}` | one agent's dependency subtree |
| `get_agent_baseline` | `get_agent_performance_baseline` | `{agentId, metric}` | current-vs-baseline |
| `get_location_status` | `get_location_status` | scope | per-location status (worst-of-instances) |
| `get_executive_health` | `get_executive_health_summary` | — | consolidated health summary |

**Excluded (curation):** `backfill_span_exceptions` is tagged `agent_tool` but is
a **mutation/backfill** — **not** exposed as an SRE read tool. All `scheduled`
sweepers, `event`, `http`, `sync` functions are excluded by the `agent_tool` gate.

## Write tool (the only write in the system)

| MCP tool | Target | Notes |
| :-- | :-- | :-- |
| `write_investigation_report` | `AiAnalysis` entity | create/update; `analysisType = external_sre_copilot`; RBAC-scoped + audited; surfaces in the TAO UI alongside the internal agents' analyses. |

`AiAnalysis` attrs today: `agentName, executionId, analysisType, inputEvidence,
outputAnalysis, timestamp, confidence`. **Gap to close (app team):** relax
`executionId` to optional, extend `analysisType` enum with `investigation`, add
optional `findingId` + `authoredBy`, and implement the `write_investigation_report`
function. Full spec (entity JSON + function JSON + checklist) →
`./SRE-Copilot-Investigation-Report-Handoff.md`.

## Candidate tools not in v1 (need functions or feature work)

- `compare_versions` (A/B — Scheduled Feature D) — add when D ships.
- `get_drift` (Feature E) — add when E ships.
- `query` (constrained aggregate read over telemetry via `/query`) — optional;
  gate carefully (RBAC + structure-only).

## Agents (context, not tools)

The app defines `sreRootCauseAgent`, `executiveSreSummaryAgent`,
`tokenOptimizationAgent`. The copilot reuses the **same functions** these agents
call (one investigation logic, two harnesses); the agents themselves are not
exposed as MCP tools in v1.

## Tool gaps — functions to expose for deeper runbooks

The runbooks work today from `get_top_platform_findings` (which surfaces **all**
finding types: reliability / cost / latency / governance / tokenEfficiency /
security) + the 10 read tools. But the **deep, type-specific** functions are **not
exposed** as `agent_tool` — flagged in the cost/latency runbooks. To power fuller
investigations, ask the app team to tag these `invocationMode: agent_tool` (or add
read wrappers):

| Function | Current mode | Powers |
| :-- | :-- | :-- |
| `analyze_latency` | scheduled | 4-bucket latency breakdown + percentile trends |
| `analyze_performance_regression` | scheduled | version-boundary regression verdict |
| `get_top_token_consumers` | sync | cost drill: top consumers |
| `forecast_costs` | scheduled | cost forecast context |
| `aggregate_costs_and_tokens` | scheduled | per-model/app cost aggregates |

## Feature dependencies (not yet built)

- **`compare_versions`** (canary-go-no-go runbook) — **Feature D** (A/B). Runbook
  runs a partial per-version-baseline flow until then.
- **`get_drift`** (drift-confirmation runbook) — **Feature E** (Drift). Runbook
  runs a partial manual-baseline flow until then.

## Open manifest decisions

- Final read-tool set (all 10 above, or a leaner core for v1?).
- Whether `get_dependency_topology` / `get_location_status` accept the scope shape
  we want (confirm against specs).
- `AiAnalysis.findingId`/`authoredBy` addition + `write_investigation_report`
  (write-path gap — see `./SRE-Copilot-Investigation-Report-Handoff.md`).
- Which of the "tool gap" functions to expose for v1 (cost/latency depth).
