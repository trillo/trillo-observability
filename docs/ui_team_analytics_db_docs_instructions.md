# UI Team — Instructions to Update Designer Docs for Analytics DB

**Audience:** the Trillo UI team (Designer / trillo-ai-ui).
**Purpose:** you added two new AppConfig properties (`analyticsDbUrl` + `analyticsDbEnabled`) in the Designer. This is the script to update the aos-docs repo so the guide matches the new UI. Paste sections 3-4 into Claude Code, drop the screenshots into the right folder, done.

**When you're ready:** the developer-facing analytics-DB chapter (§25) is already shipped and describes the feature. This work is the **Designer-facing side** — one-paragraph mentions in Part I and the AppConfig editor screenshots + copy in Part II. Small.

---

## 1. What's already in the backend

Two AppConfig fields, both shipped and in production paths as of 2026-08-26:

| Field | Type | Default | Editor label suggestion |
|---|---|---|---|
| `analyticsDbUrl` | string (max 1024) | null | *Analytics DB URL* |
| `analyticsDbEnabled` | boolean | false | *Enable Analytics DB* |

**Field semantics** (this is what the doc will explain):

- **URL blank** + platform env default set → the app uses the platform default.
- **URL blank** + no platform default → analytics-DB sink is disabled for this app; any code that requests `sink="oliverdb"` raises `AOSException`.
- **URL set** + `enabled=false` → same as disabled. The URL is remembered but not used. This is the "leave URL set, disable for experimentation" pattern.
- **URL set** + `enabled=true` → the app is ready to route telemetry to OliverDB. AOS provisions scoped keys per-invocation.

Product-facing name is **"Analytics DB"** — never "OliverDB" in editor labels. The URL field is vendor-neutral by design so a future ClickHouse / Tempo / etc. would live in the same slot.

Full backend design: `trillo-observability/docs/analytics_db_appconfig_editor_ui_handoff.md`.
Full feature docs (already shipped for developers): `trillo-ai-aos-docs/content/04-developer-guide/25-analytics-db-oliverdb.md`.

---

## 2. Screenshots to take

Take three screenshots from your Designer's AppConfig editor. Save them here:

```
trillo-ai-aos-docs/images/analytics-db/
```

Filenames (used in the doc):

1. **`analytics-db-appconfig-editor.png`** — the whole AppConfig editor screen with the new *Analytics & Observability* section visible. The two new fields should be near the middle of the frame. If the section is collapsible, take the collapsed version too as a second shot:
2. **`analytics-db-appconfig-collapsed.png`** *(optional)* — collapsed view showing where the section lives among the rest of AppConfig.
3. **`analytics-db-appconfig-filled.png`** — the same editor with example values filled in: URL = `https://trillo.us-west-2.aws.olivercloud.ai` (our shared dev tenant), toggle = ON.

If you already have the images folder in a different location, the prompt in §3 tells Claude Code to adapt.

---

## 3. Prompt to paste into Claude Code

Copy everything between the fences into a Claude Code session running against the `trillo-ai-aos-docs` repo:

```
I've added two new AppConfig properties in Trillo AI Designer's AppConfig
editor for the Analytics DB (OliverDB) integration:

  - "Analytics DB URL" (analyticsDbUrl, string, nullable)
  - "Enable Analytics DB" (analyticsDbEnabled, boolean, default false)

I've placed the screenshots at trillo-ai-aos-docs/images/analytics-db/:

  - analytics-db-appconfig-editor.png       (main view)
  - analytics-db-appconfig-collapsed.png    (optional, collapsed section)
  - analytics-db-appconfig-filled.png       (example values)

Please update three files to document these:

1. content/02-trillo-ai-designer/12-application-admin.md
   - Add a subsection under the existing AppConfig editor coverage
     describing the two new fields.
   - Use the "Analytics & Observability" section grouping (that's how
     the Designer UI groups them).
   - Embed the screenshots inline.
   - Explain the four states of the two fields (blank+off, blank+on,
     URL+off, URL+on) and what each means. Reference
     content/04-developer-guide/25-analytics-db-oliverdb.md for the
     developer story.
   - Product-facing name is "Analytics DB" -- do not say "OliverDB" in
     labels or headings. A subtle body-text mention of OliverDB as the
     currently-supported backend is fine.

2. content/product-overview.md
   - Add one short paragraph in the appropriate section (services AOS
     provides, or the runtime section, wherever observability is most
     natural) noting that AOS supports an optional columnar analytics
     store for observability telemetry, on top of Postgres. Link to
     content/04-developer-guide/25-analytics-db-oliverdb.md.

3. content/development-process.md
   - One sentence in the deploy / operate area noting that observability
     data now has a dual-store option: Postgres by default, OliverDB
     when the app enables it. Link to the same §25 chapter.

Ground your writing on:
- content/04-developer-guide/25-analytics-db-oliverdb.md -- the
  developer-facing chapter that already exists. Match its voice.
- content/glossary.md's Analytics DB section -- use those definitions.
- trillo-observability/docs/analytics_db_appconfig_editor_ui_handoff.md
  (in the other repo if visible) -- the backend design notes with
  field validation, defaults, and UX suggestions.

Then verify:
- All three files parse as valid Markdown.
- Every image path resolves under images/analytics-db/.
- Cross-links to §25 use the correct relative path.
- No mention of "OliverDB" in any editor label; the product-facing name
  is "Analytics DB" (see the handoff doc's Nomenclature section).

Commit each file separately with a short message like:
  "docs: 12-application-admin -- add Analytics DB fields"
  "docs: product-overview -- mention optional analytics store"
  "docs: development-process -- note dual-store observability option"

If anything is ambiguous, ask me before committing.
```

---

## 4. What Claude Code should produce

Rough expectation of the changes so you can eyeball the diffs:

### `12-application-admin.md`
- One new subsection under the AppConfig editor section (probably ~30 lines including one code fence for the four-state table).
- Two or three image embeds.
- One cross-link to §25.

### `product-overview.md`
- One paragraph (~5-6 lines).
- One cross-link to §25.

### `development-process.md`
- One sentence.
- One cross-link to §25.

If Claude Code produces significantly more than this on any file, ask it to trim — this is a Designer-facing addition, not a feature explainer.

---

## 5. Verification checklist (before you push)

- [ ] Screenshots are at `trillo-ai-aos-docs/images/analytics-db/` with the exact filenames from §2.
- [ ] `12-application-admin.md` renders the screenshots.
- [ ] "Analytics DB" is the label everywhere the editor is referenced; "OliverDB" appears only in body text (once or twice max) as the current backend.
- [ ] All three files link to `content/04-developer-guide/25-analytics-db-oliverdb.md`.
- [ ] Field semantics match §1 of this doc (the four states).
- [ ] `TOC.md` doesn't need updating (§25 is already listed, and 12-application-admin / product-overview / development-process are unchanged in the TOC).

---

## 6. When you're stuck

- Prompt didn't produce the four-state table clearly → paste the table from §1 of this doc directly into the file and ask Claude Code to render it cleanly.
- Voice drift (too verbose, too casual) → point Claude Code at `content/04-developer-guide/25-analytics-db-oliverdb.md` and say "match this voice."
- Cross-link paths breaking → the aos-docs convention is `content/…/NN-file.md`; verify one link resolves in the built docs site before pushing.

Ping the platform team (Slice AB/B1/C/D/E/F' owners — same person handing you this doc) if the backend behavior described here doesn't match what you're seeing in the Designer UI. Backend and doc must stay in lockstep.

---

## Change log

| Version | Date | Author | Notes |
|---|---|---|---|
| 0.1 | 2026-08-29 | Trillo platform team | Initial handoff. Ready for the UI team when the two AppConfig properties are visible in the Designer. |
