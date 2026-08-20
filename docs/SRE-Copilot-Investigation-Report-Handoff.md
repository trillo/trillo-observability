# App-team handoff: investigation-report write path (`AiAnalysis` + `write_investigation_report`)

**For:** the TrilloAgentObservability app team (appId 568).
**Why:** the TAO SRE Copilot (Claude Code plugin) needs to file an **investigation
report** back into TAO so it appears in the product UI. This is the plugin's only
write. This doc specifies the two pieces the app needs: a small extension to the
existing **`AiAnalysis`** entity, and a new **`write_investigation_report`**
function.

Design reference:
`./Enterprise_AI_Agent_Observability_SRE_Copilot_Plugin_Design.md`.

---

## 1. Class decision — extend `AiAnalysis` (do NOT create a new class)

- **Reuse the existing `AiAnalysis`.** The internal agents (`sreRootCauseAgent`,
  etc.) already write to it and it already renders in the TAO UI; the external
  copilot's report belongs in the same surface.
- **Keep the name `AiAnalysis`** (not `AIAnalysis`). Renaming for acronym casing
  (to match system `AIMessage`) is a disruptive migration and out of scope here.
- Three changes needed (all additive / backward-compatible):
  1. **`executionId` → optional** (was `required: true`). A fleet/cluster
     investigation references a **finding**, not a single execution.
  2. **`analysisType` enum** gains **`investigation`** (the external copilot's
     report type).
  3. Add **`findingId`** (optional) and **`authoredBy`** (optional) attributes.

## 2. Updated `AiAnalysis` entity JSON

Full definition with the three changes applied (new/changed parts marked with
comments — remove comments before use, JSON doesn't allow them):

```json
{
  "name": "AiAnalysis",
  "description": "Record of an AI agent's or SRE copilot's diagnostic/optimization/investigation analysis.",
  "acl": {
    "admin": ["create", "read", "update", "delete"],
    "administrator": ["create", "read", "update", "delete"],
    "owner": ["read"],
    "user": ["read"],
    "auditor": ["read"]
  },
  "attributes": [
    { "name": "id", "type": "biginteger", "persistent": true, "primaryKey": true, "systemAttr": true, "autoSequence": true },
    { "name": "createdAt", "type": "biginteger", "persistent": true, "systemAttr": true },
    { "name": "updatedAt", "type": "biginteger", "persistent": true, "systemAttr": true },
    { "name": "deleted", "type": "boolean", "persistent": true, "systemAttr": true },
    { "name": "deletedAt", "type": "biginteger", "persistent": true, "systemAttr": true },

    { "name": "agentName", "type": "string", "length": 100, "required": true, "indexed": true,
      "description": "Name of the AI agent or copilot that produced the analysis (e.g. 'sreRootCauseAgent' or 'SRE Copilot')." },

    "// CHANGED: executionId no longer required (cluster/fleet reports reference a finding)",
    { "name": "executionId", "type": "string", "length": 255, "indexed": true, "required": false,
      "description": "Optional. ID of the execution this analysis belongs to (references Execution.executionId). Null for finding/cluster-scoped reports." },

    "// NEW: findingId",
    { "name": "findingId", "type": "biginteger", "indexed": true,
      "description": "Optional. ID of the PlatformFinding (e.g. a failure cluster) this analysis is about. Set for fleet/cluster-scoped reports." },

    "// CHANGED: analysisType enum adds 'investigation'",
    { "name": "analysisType", "type": "string", "length": 50, "required": true, "indexed": true,
      "enumValues": "diagnostic,optimization,summary,investigation",
      "description": "Type of analysis. 'investigation' = external SRE copilot report." },

    "// NEW: authoredBy",
    { "name": "authoredBy", "type": "string", "length": 255, "indexed": true,
      "description": "Optional. The human user (SRE) who authored this via the copilot, when applicable. Null for fully-automated agent analyses." },

    { "name": "inputEvidence", "type": "text", "listHidden": true,
      "description": "Input evidence (structured findings/clusters/topology/baselines) the analysis was grounded in." },
    { "name": "outputAnalysis", "type": "text", "listHidden": true,
      "description": "The analysis / report body (markdown)." },
    { "name": "timestamp", "type": "timestamp", "required": true, "indexed": true,
      "description": "When the analysis was produced." },
    { "name": "confidence", "type": "decimal", "precision": 5, "scale": 2, "minValue": 0, "maxValue": 1,
      "description": "Confidence level (0.0 to 1.0)." }
  ]
}
```

**Constraint the write function enforces (not the schema):** at least one of
`executionId` / `findingId` must be present.

## 3. New function: `write_investigation_report`

Creates (or updates) an `AiAnalysis` row of type `investigation`. Follows the same
spec shape as the app's existing `get_*` functions (`invocationMode: agent_tool`,
`role` + `allowedAppRoles`, `params`, `returns`).

```json
{
  "name": "writeInvestigationReport",
  "shortDescription": "Files an SRE-copilot investigation report as an AiAnalysis (analysisType=investigation) so it appears in the TAO UI.",
  "description": "<h4>Overview</h4><p>Persists an investigation report produced by the external SRE copilot as an <code>AiAnalysis</code> row (<code>analysisType='investigation'</code>). Create, or update an existing report by id.</p><h4>Intended use</h4><p>Called at the end of an investigation runbook. Params carry the report; the function stamps agentName/timestamp and writes the row (RBAC-scoped, audited).</p><h4>Logic</h4><ol><li>Require at least one of <code>findingId</code> / <code>executionId</code>; else return bad_request.</li><li>If <code>reportId</code> is given, load + update that AiAnalysis; else create.</li><li>Set <code>analysisType='investigation'</code>, <code>agentName</code> (default 'SRE Copilot'), <code>authoredBy</code> = caller, <code>timestamp</code> = now.</li><li>Map report fields into <code>inputEvidence</code> (evidence) + <code>outputAnalysis</code> (body).</li><li>Persist; write an AdministrativeAudit record; return the id.</li></ol><h4>Entities</h4><ul><li>AiAnalysis (write)</li><li>PlatformFinding / Execution (referenced)</li><li>AdministrativeAudit (write)</li></ul>",
  "invocationMode": "agent_tool",
  "role": "user",
  "allowedAppRoles": ["admin", "administrator", "user", "owner"],
  "functionName": "write_investigation_report",
  "runtime": "python",
  "params": {
    "type": "object",
    "properties": {
      "reportId":       { "type": "integer", "description": "Optional. Existing AiAnalysis id to update; omit to create." },
      "findingId":      { "type": "integer", "description": "PlatformFinding this report is about (finding/cluster-scoped). Provide this or executionId." },
      "executionId":    { "type": "string",  "description": "Execution this report is about (execution-scoped). Provide this or findingId." },
      "rootCauseClass": { "type": "string",  "description": "Optional. CODE | DEPLOYMENT | DEPENDENCY | UNKNOWN." },
      "summary":        { "type": "string",  "description": "One-line headline of the finding." },
      "likelyRootCause":{ "type": "string",  "description": "The probable root cause (inference, stated as such)." },
      "evidenceSummary":{ "type": "string",  "description": "The structured evidence the conclusion rests on (spread, first-failing span, dependency edge, baseline delta). → inputEvidence." },
      "impactedSystems":{ "type": "string",  "description": "Optional. Impacted systems/agents." },
      "recommendation": { "type": "string",  "description": "Recommended next action." },
      "analysis":       { "type": "string",  "description": "Full report body (markdown). → outputAnalysis." },
      "confidence":     { "type": "number",  "description": "0.0–1.0 confidence." }
    },
    "required": ["summary", "analysis"]
  },
  "returns": {
    "type": "object",
    "properties": {
      "id":      { "type": "integer", "description": "AiAnalysis id written." },
      "created": { "type": "boolean", "description": "true if created, false if updated." }
    }
  }
}
```

## 4. Integration notes

- **RBAC / write access:** `allowedAppRoles` excludes `auditor` (auditors read,
  they don't file reports). Confirm against your role model; adjust if owners
  shouldn't write either.
- **Audit:** the function should write an `AdministrativeAudit` record (actor =
  caller, action = `write_investigation_report`, resource = the AiAnalysis id) so
  copilot-authored reports are traceable — consistent with the platform's audit
  posture.
- **UI surfacing:** the TAO UI already lists `AiAnalysis`; `analysisType =
  investigation` + `authoredBy` let the UI badge these as **SRE-copilot** reports
  vs. automated agent analyses. A filter/badge is a nice-to-have, not required for
  v1.
- **Content note (structure-not-payload):** the report body is the copilot's own
  *synthesized conclusion* (root cause, evidence summary, recommendation) — not
  raw telemetry payloads. If any quoted evidence includes user content, the
  existing field-level masking still applies on read.
- **Backward compatibility:** all three entity changes are additive
  (`executionId` relaxed, enum extended, two optional fields) — existing
  agent-written `AiAnalysis` rows and functions are unaffected.

## 5. Checklist for the app team

- [ ] Apply the three `AiAnalysis` changes (§2), redeploy so the schema alters.
- [ ] Implement `write_investigation_report` (§3) as a Python `agent_tool`
      function; enforce the "findingId or executionId" rule; write the audit row.
- [ ] (Optional) UI badge/filter for `analysisType = investigation`.
- [ ] Confirm the function appears in the app catalog with
      `invocationMode: agent_tool` so the SRE Copilot picks it up.
