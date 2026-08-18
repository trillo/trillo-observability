# TAO SRE Copilot — Claude Code Plugin & MCP Design

**Document Version:** 0.1 (draft)
**Companion to:** POC Requirements v1.5, SRS v1.1, Addendum + Addendum 2,
Pre-Demo (A/B/C) + Scheduled (D/E/F) feature specs. Follows the existing
**Trillo authoring plugin** (`trillo-claude-plugin`, `plugin.json` v0.4.0).

**One-liner:** publish Trillo Agent Observability (TAO) as an MCP surface so any
SRE's Claude Code becomes an **investigation copilot** — "why did agent fleet X
degrade at 2am?" as a conversation — with the platform shipping the investigation
**runbooks** that teach the agent how to investigate it.

---

## 1. Purpose & scope

TAO is a Trillo AOS application like any other (classes, functions, agents,
`/fn` · `/data` · `/query` APIs; a Trillo AI / Designer app enumerates them). This
design reuses the **authoring-plugin path** to ship a **second** Claude Code
plugin — the **SRE Copilot** — that investigates a *deployed* TAO app instead of
authoring one.

**In scope:** the plugin shell, its auth model, the secure catalog-fetch, the
runtime investigation tool surface, the single write path (investigation report),
and the runbook skills.

**Non-goals:** no true inline enforcement (AD-012 / G6 tabled); no writes to Trillo
AI (Designer) at all; no bespoke "execute arbitrary endpoint" passthrough (reuse
`/fn`).

## 2. Reuse from the authoring plugin (what is copied, not built)

The authoring plugin (`trillo-claude-plugin`) already establishes the whole shell:

| Piece | Authoring plugin | SRE Copilot |
| :-- | :-- | :-- |
| `.mcp.json` | one HTTP MCP server + OAuth (`clientId: trillo-claude-code`) | same shape; **new `clientId`**, points at the **AOS runtime MCP** |
| `.claude-plugin/plugin.json` + `marketplace.json` | manifest | copy; new name/description/keywords |
| **Skills = runbooks** | `deploy`, `functions`, `entity-model`, … (description + numbered tool steps) | **investigation runbooks** in the identical format |
| OAuth-via-MCP (plan-59) | login to Trillo AI | **login to AOS** (see §4) |

So the shell + auth + runbook mechanism are **free**. The genuinely new work is:
(a) the **runtime investigation tool surface**, (b) the **secure catalog fetch**,
(c) the **redaction boundary**, (d) the **single write path**, (e) the **runbook
content**.

## 3. Architecture — two planes, two trust levels

```
Claude Code (SRE Copilot plugin)
   │
   ├── CATALOG PLANE  ── Trillo AI (Designer) ──────────────  PUBLIC, read-only
   │      GET catalog(appId) → investigation-exposable         (allow-listed;
   │      classes / functions / agents (names, signatures,      no data, no writes)
   │      descriptions).  "What tools does app X expose?"
   │
   └── RUNTIME PLANE  ── Trillo AOS (deployed app) ─────────  OAuth + RBAC + MASKING
          read:  run investigation functions / queries          (structure-not-payload
                 over telemetry → findings, clusters, spread,     by default; raw payload
                 topology, baselines, span skeletons, logs.       gated + audited)
          write: create/update ONE InvestigationReport →
                 surfaces in the TAO UI.  (the only write)
```

**The trust split is the security design:** the **catalog** (tool names +
signatures) is non-sensitive metadata → can be public. **Data + writes** require
the AOS session (RBAC + masking). The public plane never returns data; the
authenticated plane is where everything sensitive lives.

## 4. Auth model

- **SRE authenticates directly with AOS** (not Trillo AI) — because the telemetry,
  the SRE's RBAC grants, and field-level masking all live in AOS. OAuth-via-MCP
  (plan-59) is reused; only the `clientId` and target URL change.
- The AOS session scopes every runtime tool to the caller's RBAC, so masking is
  correct **at the MCP boundary** (see §6).
- **No Trillo AI auth** — the catalog plane is public/read-only (§5). The plugin
  never holds Designer write credentials.

## 5. Catalog plane — secure, public, allow-listed

**Requirement (from the decision):** Trillo AI exposes a **public** read-only
endpoint returning an app's classes/functions/agents, **without** a shared token
if possible, and the plugin **never writes** to Trillo AI.

**How "public" is made safe — an explicit allow-list, not the whole app:**
- Trillo AI exposes only items the app **flags as investigation-exposable**
  (`sreExposable: true` on the function/class/agent, or a per-app manifest). The
  public surface is a **deliberate, minimal tool manifest** — think "an OpenAPI
  spec for the investigation tools" — not the app's full internal design.
- The response carries **names, signatures, descriptions only** — **no data, no
  secrets, no config**. So the disclosure is exactly what is *meant* to be a
  callable tool.
- Endpoint (illustrative): `GET /api/v2.0/observability/catalog?appId=<id>` →
  `{ classes:[…], functions:[…], agents:[…] }` (exposable subset).

**Why avoid the shared token:** the content is designed to be a public tool
manifest — non-sensitive by construction — so a token adds distribution/rotation
pain without protecting anything sensitive. The **allow-list is the real control**.

**Escape hatch (per-app `catalogVisibility`):** `public` (default for TAO) /
`token` / `authenticated`. Enterprises that don't want even function *names*
public can tighten to `token`/`authenticated` without changing the plugin — the
plugin tries public first, falls back to the AOS-authenticated catalog if the app
is private. Keeps the default frictionless while giving regulated tenants a knob.

*(Considered & set aside: fetching the catalog from the AOS session instead of a
public Designer endpoint — single-plane, no public surface. Rejected per the
decision to keep Designer as metadata source-of-truth and the catalog reachable
without a full runtime session; the `authenticated` visibility mode preserves that
option per-app.)*

## 6. Runtime plane — investigation tool surface (the core)

**These tools are TAO's existing functions — not new capability.** PRD §13.5
already defines the SRE Root Cause Agent's tool flow; the same functions become
MCP tools for the external copilot (the internal agent and the copilot share the
functions — do not fork the logic).

| MCP tool | Backed by | Returns |
| :-- | :-- | :-- |
| `get_finding` / `list_findings` | `platform_finding` | finding + evidence (structure) |
| `get_failure_cluster` | `FailureCluster` (Feature A) | signature, **spread**, root-cause class, impacted agents |
| `get_trace` | `otlp_span` by `trace_id` | span **skeleton** (category/timing/status), not raw payloads |
| `get_related_logs` / `get_events` | `otlp_log` / `otlp_event` | correlated diagnostics |
| `get_dependency_graph` | `agent_dependency` (AD-002) | topology + impacted systems |
| `get_baseline` | `analysis_baseline` | current-vs-baseline |
| `compare_versions` | Feature D | normalized A/B (ops + composite quality) |
| `get_drift` | Feature E | DRIFT findings |
| `query` (constrained) | `/query` over telemetry | aggregates, RBAC-scoped |
| `write_investigation_report` | §7 | the only write |

**Redaction boundary — structure, not payload, by default (the anchoring
principle):** because Claude Code egresses tool output to an external model, the
runtime tools return **investigation structure** (findings, clusters, spread,
topology, span skeletons, rates, counts, baselines) — **non-sensitive by
construction** — and *not* raw `prompt_text` / `completion_text` / tool payloads.
Raw-payload access is a **separate, RBAC-gated, audited** tool category, off by
default; when used, field-level masking is enforced server-side before the
response leaves AOS. This makes the external-model concern moot for the common
case and turns the security constraint into a feature (troubleshooting works on
incident *shape*, which is what Feature A already surfaces).

## 7. Write path — the single write: `InvestigationReport`

The **only** thing the copilot writes is an **investigation report**, back to AOS,
which surfaces in the TAO UI alongside the internal SRE agent's analyses.

- Reuse the PRD's **`ai_analyses`** concept (output of specialized SRE/analysis
  agents) — a new `AiAnalysis` row (or a dedicated **`InvestigationReport`**
  class): `agentId`/`applicationId`, `findingId`/`executionId`/`clusterId`,
  `summary`, `likelyRootCause`, `evidenceSummary`, `impactedSystems`,
  `recommendations`, `confidence`, `analysisAgentType = EXTERNAL_SRE_COPILOT`,
  `invocationType = USER`, `authoredBy` (the SRE), timestamps.
- A single narrow AOS function/endpoint (`write_investigation_report`) —
  create/update, RBAC-scoped, **audited** (administrative-audit). No other AOS
  writes; **no Trillo AI writes**.
- Result: the SRE's Claude Code session ends with a durable, attributed report in
  the product — closing the loop from conversation → recorded finding.

## 8. Runbooks = investigation skills (the differentiator)

Same format as the authoring plugin's skills (`description` + numbered tool
steps). The platform ships the playbooks so the agent knows *how* to investigate:

- **`reliability-incident-triage`** — `list_findings(status=ERROR)` →
  `get_failure_cluster` (read spread → CODE vs DEPLOYMENT vs DEPENDENCY) →
  `get_dependency_graph` for impacted systems → `get_trace` skeleton of an
  exemplar → draft `write_investigation_report`.
- **`cost-spike-investigation`** — cost rollups → offending app/agent/model →
  `compare_versions` if a version boundary → report.
- **`latency-regression-drilldown`** — latency percentiles → bottleneck category →
  exemplar trace → baseline compare.
- **`drift-confirmation`** — `get_drift` → recent-vs-baseline → provider-vs-input.
- **`canary-go-no-go`** — `compare_versions` (B vs A, 24h, normalized + quality) →
  rollout verdict → report.

These encode the **same tool sequences** the internal SRE Root Cause Agent uses
(§13.5) — one investigation logic, two harnesses.

## 9. New-to-build vs. reused

**Reused (free):** plugin shell (`.mcp.json`/manifest), OAuth-via-MCP, skills-as-
runbooks mechanism, TAO's investigation functions, `ai_analyses` concept, RBAC +
masking.

**New to build:**
1. **Runtime app-MCP surface** on AOS that exposes the app's **allow-listed**
   functions/queries as MCP tools, OAuth+RBAC-scoped, **structure-not-payload**
   default. (Generic AOS capability — any app benefits; likely the proposal's
   "TOP Skill MCP Server".)
2. **Public catalog endpoint** on Trillo AI (allow-listed, `catalogVisibility`).
3. **`sreExposable` flag** (function/class/agent) + the allow-list plumbing.
4. **`write_investigation_report`** function + `InvestigationReport`/`AiAnalysis`
   surfacing in the TAO UI.
5. **The SRE Copilot plugin** (manifest + `.mcp.json` + the runbook skills).

## 10. Open items / decisions

- **Catalog default visibility** — `public` (frictionless, TAO's own use) vs.
  `authenticated` default for enterprise tenants. Recommend `public` default +
  per-app override.
- **Raw-payload posture** — confirm "structure-only default, raw gated+audited" is
  acceptable for the SRE flow (it should be — infra troubleshooting rarely needs
  raw prompts). This is the egress-safety linchpin.
- **Tool sourcing** — does the AOS runtime MCP derive its tool list from the
  deployed function registry directly, or from the (Designer) catalog? Both work;
  pick one source of truth (lean: AOS runtime registry for *execution*, Designer
  catalog for *discovery/preview*).
- **Report class** — extend `ai_analyses` vs. a dedicated `InvestigationReport`
  class. Lean: reuse `ai_analyses` (already surfaces in the UI).
- Need the **TAO appId + its function/agent/class list** to map §6 tools to real
  signatures and finalize the runbook steps.
