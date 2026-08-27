# UI handoff — AppConfig editor: analytics-DB fields

**Audience:** the Trillo AI UI team (**Trillo AOS Designer** — per the 2026-07-31 rename; repo still `trillo-ai` / `tcs-ui`).
**Why:** the OliverDB / analytics-DB integration adds two new `AppConfig` fields (Slice AB, shipped 2026-08-26 to `develop` on tcs-metadata + tcs-core + trillo-aos). The Designer's **AppConfig editor** needs a small addition so administrators can point an app at an analytics store and turn the sink on. Nothing else in this feature ships until the editor can capture the values.
**Companion docs:**
- `aos_oliverdb_integration_plan.md` — full integration design (§8.1 Slice AB is the piece already built).
- `oliverdb_improvements.md` — OliverDB team's roadmap items this sits on.
- `analytics_db_appconfig_editor_ui_handoff.md` — this doc.

---

## 1. What's changing on the backend (already shipped)

Two additive columns on `AppConfig` (source of truth: `tcs-metadata/app-classes/AppConfig.json`):

| Field | Type | Nullable | Default | Persistence |
|---|---|---|---|---|
| `analyticsDbUrl` | string (max 1024) | yes | — (null) | `copyIfPresent` — a blank on redeploy preserves the live value |
| `analyticsDbEnabled` | boolean | no | `false` | Always-write — a redeploy CAN flip it back to false (matches `multiTenant` / `authGuestEnabled` precedent) |

Both are `platform: true` — they ship in the deploy payload from Designer to AOS and are persisted by `DeployAppMetadata.bootstrapAppConfig`. No new deploy endpoint; they ride the same `POST /api/v2.0/deploy/app` payload alongside the other `AppConfig` fields.

## 2. What we're asking the UI to add

**One small section on the AppConfig editor screen** in Trillo AOS Designer — same screen that already edits `frontendUrl`, `logoUrl`, `multiTenant`, and the other AppConfig fields. Two controls:

### 2.1  Analytics DB URL

- **Label:** *Analytics DB URL*
- **Control:** single-line text input.
- **Placeholder:** `https://your-tenant.us-west-2.aws.olivercloud.ai`
- **Help text (inline, small):**
  > The columnar analytics store this app writes telemetry to (currently OliverDB). Leave blank to inherit the platform-level default. Vendor-neutral field name — same URL slot works for any future analytics backend Trillo integrates.
- **Validation:**
  - Optional. Blank is valid (means "inherit env default, or disable").
  - When non-blank: must parse as an `https://` URL. Reject `http://` (analytics traffic is never over cleartext).
  - Trim trailing slash on submit (Designer-side, so redeploys don't produce spurious diffs).
- **Default on new app:** blank.
- **Diff behavior:** blank → non-blank is an edit (enables per-app override); non-blank → blank is an edit (clears the override — falls back to platform default).

### 2.2  Enable Analytics DB

- **Label:** *Enable Analytics DB*
- **Control:** toggle / switch (off by default).
- **Help text (inline, small):**
  > When on, functions in this app can route telemetry to the Analytics DB by requesting `telemetrySink: "oliverdb"`. When off (default), telemetry stays in Postgres regardless. Belt-and-braces off switch — even if the URL above is set, this must be on for the pod to route to the Analytics DB.
- **Validation:**
  - No extra rules. Enabling with no URL and no platform default resolved is not an error at edit time — it becomes a runtime "analytics sink unavailable" that shows up in the pod's own diagnostics.
- **Default on new app:** off (`false`).
- **Interaction with URL field:** independent — do NOT auto-disable when URL clears, and do NOT auto-enable when URL is filled. The two knobs are deliberately separate.

### 2.3  Grouping and placement

- **Section header suggestion:** *Analytics & Observability* (a new subsection). It anchors well before the `Chat` and `Auth` sections which are already grouped.
- Order the two controls URL first, toggle second.
- If the editor uses collapsible sections, this subsection can start collapsed by default — most apps won't use it initially.

## 3. Backend contract (what to send)

Same payload shape as every other AppConfig field. Add two keys to the AppConfig object nested inside the deploy payload:

```json
{
  "app": {
    "name": "trillo-agent-observability",
    "appConfig": {
      "frontendUrl": "https://obs.customer.com",
      "multiTenant": false,
      "analyticsDbUrl": "https://acme.us-west-2.aws.olivercloud.ai",
      "analyticsDbEnabled": true,
      "// ... other AppConfig fields": "..."
    }
  }
}
```

Send both fields on **every deploy**, even when unchanged — same as `multiTenant`, `authGuestEnabled`, etc. Sending only-changed values is not a supported shape.

**Type discipline:**
- `analyticsDbUrl`: JSON string, or `null` / omitted for blank. Empty string `""` is treated the same as `null` on the AOS side; either is fine.
- `analyticsDbEnabled`: JSON boolean `true` / `false`. AOS also accepts `"true"` / `"false"` strings from the Designer editor, but send a real boolean when possible.

## 4. Read side — where the persisted value comes back

The Designer's editor also **reads** current values so administrators see what's live. Two paths:

- **Fresh app being edited before first deploy:** no persisted values yet; the editor uses its own local state (blank / false).
- **Already-deployed app:** Designer typically renders the AppConfig editor from the same source-of-truth JSON it ships to AOS (its own store), not from AOS's `AppConfig` row. If that's the case, no read-side change is needed — the two new keys just flow through Designer's own storage.

If Designer *does* read live values from AOS for this screen (via `GET /api/v2.0/md/app-config`), the two new fields are already included in the composer projection — no server-side change needed. Confirm with the API surface you use.

## 5. Verification

To smoke the whole path end-to-end:

1. Open the AppConfig editor for a test app.
2. Fill `Analytics DB URL` = `https://trillo.us-west-2.aws.olivercloud.ai` (our shared dev tenant).
3. Toggle `Enable Analytics DB` ON.
4. Deploy the app.
5. On the AOS instance, tenant-0 admin can verify with:
   ```
   GET /api/v2.0/admin/oliverdb/health?appId=<the-app-id>
   ```
   Successful response includes `{ "url": "https://trillo...", "upstream": { "keys": [...] } }` — the URL round-tripped from Designer through deploy to AOS's platform admin surface.
6. Toggle OFF, redeploy, re-hit `/health` → the URL still resolves (falls back to `tcs.analytics.db.default.url` env-level), but the pod-launch path in Slice C will refuse OliverDB routing because the enable flag is off.

If step 5 fails with `ANALYTICS_DB_NOT_CONFIGURED`, the field didn't reach AOS — either the deploy payload didn't include the key, or `DeployAppMetadata.bootstrapAppConfig` wasn't loaded (needs the trillo-aos develop @ 8f3c310 or later).

## 6. Nomenclature notes

- **Product-facing name is *Analytics DB***, not "OliverDB." The field name is deliberately vendor-neutral because the same knob will route to ClickHouse / Tempo / etc. in future integrations. All UI copy should say *Analytics DB* — never mention OliverDB by name in the editor. That keeps the UI stable when we add a second backend.
- If space allows, a subtle badge or footer text like *"Powered by OliverDB (Aug 2026)"* is fine, but not in the labels themselves.

## 7. Out of scope for this handoff

- Bulk / import editors, environment-scoped overrides, or a separate "Analytics" tab. All of that is v2 — the two-field addition on the existing AppConfig editor is the whole Slice AB UI ask.
- Any read-side integration with OliverDB from the Designer UI. Designer stays a pure authoring surface; runtime observability queries happen in the deployed app UI, not in Designer.
- Live-testing the URL from the editor (a "Test connection" button). Health is available via the tenant-0 admin endpoint above; putting it in the per-app editor would need per-app admin scoping we haven't designed yet.

## 8. Contacts

- Backend / API questions: platform team, this integration's design owner (this session's counterpart).
- Full integration design + slice plan: `trillo-observability/docs/aos_oliverdb_integration_plan.md`.
- OliverDB team's wire contract (for context, not something the UI touches): `trillo-observability/docs/oliverdb_onboarding.md` + `oliverdb_onboarding_admin.md`.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-08-26 | Trillo platform team | Initial handoff. Slice AB backend already shipped; this is the Designer editor addition that unblocks the field from reaching AOS at deploy time. |
