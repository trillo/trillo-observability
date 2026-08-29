# OliverDB — Update Plan for `trillo-ai-aos-docs`

**Document Version:** 0.1 (draft)
**Status:** For review before Slice I' work begins.
**Companion to:**
- `oliverdb_refactor_plan.md` — the code-side refactor plan.
- `app_team_oliverdb_guide.md` — the standalone developer guide (this plan is about integrating that guide's content into the formal docs repo).
- `oliverdb_improvements.md` — the OliverDB team wishlist.

## 1. Purpose

`trillo-ai-aos-docs` is the formal developer-facing documentation set for Trillo AI + AOS. It has both a "Designer" audience (Part II) and a "Claude Code developer" audience (Part III + IV). The Slice AB→F' work introduced a new subsystem (analytics-DB / OliverDB) that isn't mentioned anywhere in that repo yet. This plan enumerates every file that needs a touch and what the touch is.

## 2. Scope decision

Two levels of scope you can pick from:

### 2.1  Minimal scope (recommended for the first pass)
Just the developer-facing chapters. Enough for a developer using Claude Code or writing functions directly to discover the API, understand the sink model, and follow the copy-paste patterns.

- **New chapter** in Part IV/Operate: `04-developer-guide/25-analytics-db-oliverdb.md`
- **Update** Part III Claude Code: `03-generation-flow.md` + `05-reference.md`
- **Update** Part IV/Operate: `04-developer-guide/18-tasks-logs-observability.md` (one cross-ref)
- **Add** glossary entries: OliverDB, sink, dispatcher pattern

Total: 1 new file, 4 modified files. Estimate: ~half a day of writing.

### 2.2  Full scope (do once Designer UI ships the AppConfig fields)
Adds the Designer-facing surface. Blocked on the Designer UI ship for the two new `AppConfig` fields (see `analytics_db_appconfig_editor_ui_handoff.md`).

- Everything in 2.1 plus:
- **Update** Part II Designer: `02-trillo-ai-designer/12-application-admin.md` — document the two new AppConfig editor fields when they ship.
- **Update** Part I: `product-overview.md` — one paragraph on OliverDB as an optional analytics sink.
- **Update** Part I: `development-process.md` — mention observability data now has a dual-store option.
- **Add** FAQ entries: "When do I need OliverDB?" and "How do I turn it on for my app?"

Total: 5 modified files on top of 2.1.

**My recommendation: ship 2.1 now, hold 2.2 pending the Designer UI ship.** The dev-facing chapters are the immediate need; the Designer-facing bits should wait for the UI so the doc and the product ship together.

The rest of this plan details the 2.1 scope; 2.2 items are listed in §7 for when the moment comes.

## 3. New chapter — `04-developer-guide/25-analytics-db-oliverdb.md`

Slotted into Part IV / Operate (after §24 user-management, at position 25). Sits alongside §18 (tasks/logs/observability) but distinctly focused on the analytics-DB integration.

**Suggested structure** (mirrors `app_team_oliverdb_guide.md` but tuned to the aos-docs voice):

1. **What it is** — one paragraph. OliverDB is an optional columnar analytics store; Postgres remains the default. Rule of thumb ("if you'd write UPDATE, it's Postgres").
2. **Turning it on for your app** — `AppConfig.analyticsDbUrl` + `analyticsDbEnabled`. Two settings, both live in AppConfig. Editable in Designer / at deploy time. Both must be truthy for OliverDB writes to succeed.
3. **Writing telemetry from a function** — `ctx.telemetry.emit_span/log/event(...)`, the `sink` kwarg, the loud-fail semantics. Copy-paste pattern for the `params.get("telemetrySink", "postgres")` idiom.
4. **Producer discipline** — `resource_attributes["service.namespace"] = str(ctx.app_id)`; `gen_ai.*` semconv on LLM spans; `execution_id` kwarg for correlation.
5. **Reading telemetry from a function** — `ctx.telemetry.query(sql)`; the dispatcher pattern (`_handler_postgres` + `_handler_oliverdb`); shape-parity requirement.
6. **OliverDB SQL notes** — one table `t`; attrs-key access; no JOINs/CTEs; `time_bucket`; `percentile_cont`.
7. **Failure modes with fixes** — same five failure messages as `app_team_oliverdb_guide.md §5`, tuned to the aos-docs voice.
8. **What's not in scope yet** — logs/events tables pending; metrics table not confirmed; cross-tenant SQL future; no browser-direct.
9. **See also** — pointers into `03-claude-code/03-generation-flow.md`, `18-tasks-logs-observability.md`, the standalone `app_team_oliverdb_guide.md`.

**Rendering**: source is essentially `app_team_oliverdb_guide.md`, rewritten to fit the aos-docs prose style, with links updated to the aos-docs internal file paths and the "see also" section pointing at neighbor chapters.

**Length**: ~250-350 lines of markdown. In line with the existing Part IV chapters.

## 4. Existing chapters to touch

### 4.1  `03-claude-code/03-generation-flow.md` — mention `ctx.telemetry`

The current chapter walks through the generation flow (requirements → spec → entity model → functions → agents → deploy → test). Under the **functions** step (or the "post-deploy guidance / local toolkit" note) add:

- A sentence noting that `ctx.telemetry` is one of the toolkit's ctx.* modules and that Claude Code's `functions` skill has an inline callout for the sink kwarg + producer discipline.
- One-line pointer to the new §25 for full detail.

**Insert location**: existing sentence "Ground on the installed aos_toolkit typed API…" is the natural home for this — same treatment as `ctx.email` / `ctx.sms` / etc. Add `ctx.telemetry` to the list.

**Delta**: ~5 lines.

### 4.2  `03-claude-code/05-reference.md` — add to tool + toolkit catalogs

This is the "lookup companion" chapter. Add:

- To the `ctx.*` toolkit table (if there is one — need to open the file to confirm): `ctx.telemetry` row with a one-line description and pointer to §25.
- Under "recipes" — a short entry: "Emit a span to OliverDB — see the app-team guide + §25 for the sink kwarg pattern."

**Delta**: ~10 lines.

### 4.3  `04-developer-guide/18-tasks-logs-observability.md` — cross-ref the new §25

This chapter today covers task events, conversation logs, SSE, troubleshooting. The two topics touch — logs and telemetry can co-exist. Add:

- One paragraph explaining the distinction: `ctx.task.log` is for per-task console + `TaskEvent` rows (troubleshooting per-invocation), while `ctx.telemetry.emit_span/log/event` is for OTel-shaped observability data flowing into Postgres or OliverDB.
- Cross-reference: "For the analytics-DB integration and the `ctx.telemetry` API, see §25."

**Delta**: ~15 lines.

### 4.4  Glossary — add entries

`content/glossary.md`. Three new entries:

- **OliverDB** — the columnar analytics store used as an optional destination for OTLP telemetry (spans/logs/events). Row-pinned per-app via scoped keys; queried with SQL via `ctx.telemetry.query`. See §25.
- **Analytics DB sink** — the choice between `"postgres"` (default) and `"oliverdb"` on a `ctx.telemetry` emit call. See §25.
- **Dispatcher pattern** — a Trillo convention for reader functions that support both Postgres and OliverDB: one `handler` routes on `params["telemetrySink"]` to `_handler_postgres` or `_handler_oliverdb`, both returning the same shape. See §25.

**Delta**: ~15 lines.

## 5. Slice I' — work order

Do the writes in this order so cross-links are always valid:

1. **Draft §25 first** (the new chapter). This is the anchor everything else points at.
2. **Add glossary entries**. Cross-links from other chapters resolve to real glossary rows.
3. **Update §18** (tasks-logs-observability). Cross-ref the new §25.
4. **Update §03-claude-code/03-generation-flow.md**. Point at §25 and mention `ctx.telemetry` in the toolkit list.
5. **Update §03-claude-code/05-reference.md**. Add to catalog + recipes.

Each is a small, mechanical change after §25 lands. Estimate: 3-4 hours end-to-end.

## 6. Voice + style conventions to preserve

Reading a couple existing chapters (18-tasks-logs-observability, 03-functions), the aos-docs style is:

- **Author's voice**, second person. "You emit a span…"
- **Runnable code blocks**, not just examples.
- **Cross-references** with the exact `content/…/NN-file.md` path.
- **Blockquote callouts** for platform quirks / gotchas.
- **Numbered sections** rooted at `##`.
- **Sources / audience / status HTML comment** at the top.

The new §25 will follow the same conventions.

## 7. Deferred (Slice I'' — post-Designer-UI-ship)

When the Designer UI adds the two new AppConfig fields (per `analytics_db_appconfig_editor_ui_handoff.md`):

- **`02-trillo-ai-designer/12-application-admin.md`** — document the two new editor fields. Slot into the existing AppConfig section (§AppConfig subsection, mid-chapter). Include a screenshot when the UI ships.
- **`product-overview.md`** — one paragraph on OliverDB as an optional analytics sink for AI observability workloads.
- **`development-process.md`** — mention that observability data has a dual-store option now.
- **FAQ**: at least one entry ("When should I turn on OliverDB for my app?") in an appropriate FAQ file.

None of these are blocked in principle — could be written now — but I'd rather write them alongside the UI screenshots so the doc doesn't drift.

## 8. What I'm not proposing to do

- **No new "Part V" or major restructuring** — the docs already have logical homes for everything. Adding chapters in place fits the existing structure.
- **No auto-generated content** — the source of truth for OliverDB SQL is the OliverDB onboarding docs we've copied into `trillo-observability/docs/`; the aos-docs chapter references that rather than duplicating.
- **No API reference generation** — `04-developer-guide/05-api-reference.md` points at `app.trillo.ai/docs` (an auto-generated OpenAPI page). The new `ctx.telemetry` toolkit surface will be picked up by the auto-generation pipeline; no manual entry needed there.

## 9. Open questions before Slice I' starts

1. **File numbering**: is §25 the right slot, or does something else need to slide to make room? Check with whoever owns `TOC.md`.
2. **Voice**: do we want the new chapter to be **prescriptive** ("do this") or **reference-style** ("here are all the options")? My lean: prescriptive, since it mirrors the `app_team_oliverdb_guide.md` shape.
3. **Cross-linking to `trillo-observability/`**: the aos-docs repo doesn't currently link out to other Trillo repos. Do we host the standalone `app_team_oliverdb_guide.md` inside aos-docs too (a copy), or link out? My lean: keep the standalone in `trillo-observability/` and reference the aos-docs chapter as the canonical developer-facing doc. Two docs, one voice per audience.

Answer these three and I'll start on §25.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-08-29 | Trillo (via Claude Code session) | Initial plan. 2.1 minimal scope proposed for the first pass; 2.2 full scope deferred to Designer UI ship. |
