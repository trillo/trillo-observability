# OliverDB — OPERATOR onboarding for tenant `default`

You hold an **admin** key (`trillo-admin`). This doc grounds you to mint access for OTHER agents/users by authoring least-privilege **policies**. Read the mental model first — it changes how you write policies.

## Mental model: policy REWRITES, it doesn't just reject

The scope travels with the KEY, not the connection. Before any query runs, the key's `Policy` runs `enforce(query)` which either REWRITES the query to fit the scope or DENIES it. "Rewrite" is the important half:

- an over-wide time window is **clamped down** to `max_window_secs`;
- an unconstrained value-restricted dim gets `IN <allowed>` **injected**;
- `inject_where` predicates are **AND-ed on** (row-level security);
- `top_k`/projection `limit` are **capped** to `max_groups`.

So a key scoped to `region=eu, ≤24h-wide window, no user_id` can be handed an untrusted agent that asks for `everything, all time, by user_id` and it comes back correctly narrowed — not an error, not a leak. You author the boundary once; the engine holds it every query.

## The Policy — every field, what it does, and the gotcha

A `Policy` is JSON; an empty policy `{}` is allow-all (no-op). Fields:

| field | type | effect (enforcement) | gotcha |
|---|---|---|---|
| `inject_where` | `[Predicate]` | RLS: each predicate is AND-ed onto every query. An `Eq` a caller re-states differently is DENIED (can't probe outside scope). | An `Eq` pin means the caller may only re-state the SAME `Eq`, not `In`/`Ne` — else clean denial. |
| `allow_values` | `{dim: [vals]}` | Per-dim value allow-list. A query constraining the dim outside the set is denied; an UNconstrained dim gets `IN <set>` injected. | Restricted dims accept ONLY `Eq`/`In` within the set — `Ne`/`LIKE`/regex are rejected (they can match outside). |
| `allow_dims` | `[dim]` | Columns the key may reference AT ALL (group_by/where/filter/select/countIf/text). Empty = any. | Covers EVERY position — a redacted-adjacent column in a filter tree or projection is caught too. |
| `allow_measures` | `[col]` | Numeric columns the key may aggregate. Empty = any. | Guards an EXPLICIT measure; the schema's DEFAULT measure (if any — see the schema section) is exempt. Only has observable teeth with ≥2 numeric columns. Via SQL, a non-numeric target trips the planner's type-check FIRST (a type error, not a policy error) — the DSL path gives the clean `measure not permitted` denial. |
| `allow_aggs` / `deny_aggs` | `[name]` | Aggregate allow-list (if set, ONLY these) then deny-list. Names: `count`,`sum`,`avg`,`min`,`max`,`countIf`,`countDistinct`,`p50`…,`argMin`,`argMax`. | A rule pack is gated as `countIf`. |
| `redact_dims` | `[dim]` | A redacted column may NOT be referenced ANYWHERE — not group_by, not where, not countIf/text. | Stronger than allow_dims: it blocks even a WHERE filter, because differential counts leak the distribution. |
| `max_window_secs` | `int?` | Caps the WIDTH of `[from,to]` (clamps DOWN, keeps `to`, moves `from`). | **Not a recency floor** — it limits span, NOT age. A caller can request an ANCIENT N-second-wide window and it passes untouched; there is no policy field that floors `from` to "now − N". If you need "only recent data", that's not expressible today (see limitations). Also: an over-wide `to` can clamp `from` PAST the data → 0 rows, HTTP 200 (use a `to` near the real timestamps). |
| `max_groups` | `int?` | Caps `top_k` AND raw-projection `limit` — bounds how much detail leaves. | Applies to bottom-K too; the count is capped in either direction. |
| `focus` | `str?` | Free-text lens — guidance ONLY, not enforced. Surfaced in the manifest so the agent knows its intended job. | Never a security control; purely advisory. |

**Attrs columns and policy** (semi-structured `Attrs` columns, filtered as `attrs.<key>`):

- `allow_dims`/`redact_dims` scope on the BASE column AND accept full key names: redacting `attrs` blocks ALL key access (filters AND `GROUP BY attrs.<key>` — group keys reveal values, so grouping is governed identically); redacting `attrs.flow` blocks just that key. An allow-list entry of `attrs` permits every key; `attrs.flow` permits exactly one. Redaction always wins over allowance.
- `inject_where` WORKS with attr keys — pin `{"dim": "attrs.tenant", "op": "Eq", "values": ["acme"]}` for row-level scope on a semi-structured field. Enforcement differs from keyword dims: there is no re-statement conflict check (a caller's own `attrs.tenant` constraint lives in the filter tree), but the AND-backstop holds — the caller's constraints INTERSECT the injection, so scope can only narrow, never widen. A cross-scope probe returns 0 rows rather than a denial.
- `allow_values` does NOT apply to attr keys (it scopes real dim columns only).

## Enforcement order (how a query is rewritten)

`enforce()` runs these in order; the first failing check DENIES with a reason string, otherwise the query is rewritten and a list of applied notes is returned:

1. **column allow-list** — every referenced column ∈ `allow_dims`
2. **redaction** — no referenced column ∈ `redact_dims`
3. **aggregate allow/deny**
4. **allow_values** — validate caller constraints, or inject `IN <allowed>`
5. **inject_where** — pin RLS predicates (conflict = deny)
6. **window clamp** — narrow an over-wide `[from,to]`
7. **detail cap** — clamp `top_k` / projection `limit` to `max_groups`

Design it least-privilege: grant the minimum `allow_dims`/`allow_measures`, redact PII dims, pin the tenant/row scope in `inject_where`, cap the window and detail. The enforcement is the same in single-instance AND cluster mode (validated by `rbac_harness`, a 6000-pair independent decision oracle), so a policy you author holds across the fleet.

## Schema lifecycle — create first, evolve carefully (admin)

| method + path | does |
|---|---|
| `GET /v1/schema` | read the current schema |
| `POST /v1/schema` | **initial create** (fresh tenant, admin bearer). Columns FREEZE at first ingest — a POST after data lands is rejected. |
| `POST /v1/cube` · `DELETE /v1/cube/:name` | add/drop a cube rollup AFTER data exists (body = the CubeDef shape below) |
| `POST /v1/sketch` · `DELETE /v1/sketch/:name` | add/drop a percentile-sketch rollup after data exists |
| `POST /v1/index` | add an index to an existing column (backfills existing files) — body `{"column": "…", "kind": "LabelBitmap"}` |

Create body (`POST /v1/schema`):
```json
{
"time_unit": "Millis",
"columns": [
{"name": "service",  "ty": "Keyword", "indexes": ["LabelBitmap"]},
{"name": "user_id",  "ty": "Id",      "indexes": ["Bloom", "LabelBitmap"]},
{"name": "latency",  "ty": "Float",   "indexes": ["ZoneMap", "AggCube", "Sketch"]},
{"name": "body",     "ty": "Text",    "indexes": ["Fts"]}
],
"cubes":    [{"name": "c1", "dims": ["service"], "measures": ["latency"], "tiers_secs": [60]}],
"sketches": [{"name": "s1", "dims": ["service"], "measures": ["latency"], "tiers_secs": [60]}]
}
```

GOTCHAS:
- **Every ingested row needs a `ts`** (in the schema's `time_unit`) — the primary sort/prune key. No natural time? Send a synthetic ordinal, or run the server with `OLIVERDB_AUTO_TS=arrival` and OMIT `ts` (arrival time is stamped server-side, which prunes/compacts BETTER than a fabricated constant). Without one or the other, ingest is rejected `400 row missing ts`.
- **Design columns BEFORE ingesting** — columns are immutable once the first row lands; only cubes/sketches/indexes evolve after (the endpoints above), never columns. Until the first row lands you may re-`POST /v1/schema` freely to fix mistakes (full replace).
- **There is NO schema delete.** No `DELETE /v1/schema`, no tenant wipe, no truncate — removing a schema (and its data) is an operational act on the tenant's data directory, not an API call. Retention ages data out but never touches the schema.
- Omit a column's `indexes` to get sensible defaults by type: Keyword→LabelBitmap · Id→Sparse,Bloom · Text→Fts · Float→ZoneMap,AggCube,Sketch · Int→ZoneMap · Bool→LabelBitmap.
- **At most ONE Text column per schema** (the FTS sidecar indexes a single text field; a second is rejected at `POST /v1/schema`) — make additional string fields `Keyword` (short/enumerable values) or fold the prose into the one Text column.
- Cubes/sketches are ingest-time rollups powering the dashboard fast paths — declare the dims you'll `group_by` (Keyword/Bool only) and numeric measures; `tiers_secs: [60]` is the usual tier.

## Non-time-series data (spatial · imaging · any keyed fact)

The engine is time-keyed, but the data need not BE a time series — spatial, imaging, genomic, and event data model cleanly with these conventions:

- **`ts` is mandatory but may be synthetic** — an acquisition time if you have one, else a batch/ingest ordinal, else `OLIVERDB_AUTO_TS=arrival` (omit `ts`, server stamps it).
- **Long-tail attributes → `Attrs`** — one semi-structured JSON-object column (e.g. `attrs`, `resource_attrs`) absorbs fields senders add over time, so the schema NEVER needs a column change. Keys are filterable as `attrs.<key>` (Eq/In index-served via the label sidecar, cardinality-guarded); columns stay for the stable core you group/aggregate on.
- **Spatial coordinates → `Float` + `ZoneMap`** — makes bounding-box ROI queries fast (`WHERE x BETWEEN … AND y BETWEEN …`). RANGE filtering only: no polygons, nearest-neighbor, or spatial joins (no geometry engine) — ideal for rectangular regions.
- **Categorical facets → `Keyword` + `LabelBitmap`** (+ `AggCube` on ones you `group_by`): image / sample / marker / class / run ids.
- **Per-object id → `Int`, or `Id`** for exact single-object needle lookup (Sparse+Bloom).
- **Measurements → `Float`/`Int` + `ZoneMap` + `AggCube` + `Sketch`** — intensities, areas, scores; `Sketch` gives percentile distributions per facet.
- **Cross-tab rollups → cubes + sketches** over the facet combinations you'll dashboard.
- **Similarity → `Vector(N)`** for per-object embeddings (morphology/expression) → GPU batch similarity search.

Worked example (single-cell imaging): dims `image_id`/`marker`/`cell_type` (Keyword), coords `centroid_x`/`centroid_y` (Float+ZoneMap), measures `mean_intensity`/`cell_area` (Float/Int + Sketch), id `cell_id` (Int), cubes over image × marker × cell_type.

## This tenant's schema (author policies against REAL columns)

| Column | Type | Role | Redaction candidate? |
|---|---|---|---|
| `trace_id` | Id | dimension (group/filter) | **yes — likely PII** |
| `span_id` | Id | dimension (group/filter) | **yes — likely PII** |
| `parent_span_id` | Id | dimension (group/filter) | **yes — likely PII** |
| `service_name` | Keyword | dimension (group/filter) | — |
| `span_name` | Keyword | dimension (group/filter) | — |
| `span_kind` | Keyword | dimension (group/filter) | — |
| `status_code` | Keyword | dimension (group/filter) | — |
| `duration_us` | Int | measure (aggregate) | — |
| `input_tokens` | Int | measure (aggregate) | — |
| `output_tokens` | Int | measure (aggregate) | — |
| `resource_attrs` | Attrs | attributes (semi-structured key=value) | — |
| `attrs` | Attrs | attributes (semi-structured key=value) | — |

Dimensions: `trace_id`, `span_id`, `parent_span_id`, `service_name`, `span_name`, `span_kind`, `status_code` · Measures: `duration_us`, `input_tokens`, `output_tokens`
- **Time unit: `Millis`** — `ts`, `from_ms`, `to_ms` are in this unit (relative to the data's REAL timestamps, not wall-clock). Query bounds also accept DATETIME LITERALS — SQL `ts >= '2024-01-01T12:00:00Z'` (RFC3339 / naive-UTC / date-only / `TIMESTAMP '…'` / `to_timestamp(secs)`), DSL `from`/`to` string twins — converted server-side to this unit; integers are NEVER unit-guessed. NOTE: `max_window_secs` is ALWAYS in **seconds** regardless of the schema unit (7 days = 604800; 30 days = 2592000).
- **Data range: empty** (no rows yet) — queries return nothing until data is ingested.
- **Default measure: none** — every aggregate must name its measure explicitly, so `allow_measures` has no default-path exemption to worry about here.

## Admin API (all under your admin key's bearer token)

Base: `https://oliverdb-trillo.tnt-trillo.svc.cluster.local:8080`

| method + path | does |
|---|---|
| `POST /v1/admin/keys` | mint a key — body below; returns `{id, token, ...}`. The token is shown ONCE. |
| `GET /v1/admin/keys` | list keys (id, principal, admin, and each key's `manifest`) |
| `PUT /v1/admin/keys/:id` | update a key's policy/flags |
| `DELETE /v1/admin/keys/:id` | revoke a key |
| `GET /v1/admin/roles` · `PUT /v1/admin/roles/:name` · `DELETE …` | role CRUD (a named, reusable policy shape) |

Mint request body (`pin` keys MUST be real columns of THIS schema — an unknown column fails closed):
```json
{
  "principal": "reporting-agent",
  "can_write": false,
  "admin": false,
  "policy": { …see recipes… },
  "role": null,               // OR: mint from a named role — supplies the BASE policy
  "pin": {"trace_id": "<value>"}   // each entry becomes an `Eq` predicate AND-ed into inject_where
}
```
**role + pin merge:** `role` supplies the base policy/flags; each `pin` entry is layered ON TOP as an AND-ed `Eq` — BOTH survive (author the shape once as a role, mint many keys each pinned to one slice). `PUT /v1/admin/roles/:name` takes `{"policy": {…}, "can_write": bool, "admin": bool}`.

**Response conventions:** success → JSON (a mint returns `{"id", "token", …}`, token shown ONCE). A policy denial → **HTTP 400, plaintext body** (e.g. `policy: column 'user_id' is redacted by policy`) — read the raw body, not a JSON `.error` field.

## Recipes (grounded in this schema)

**Read-only viewer, one slice, ≤24h-wide window, no PII, capped detail** — the default untrusted grant (`max_window_secs` caps SPAN not recency — see limitations):
```json
{
  "principal": "viewer-eu", "can_write": false, "admin": false,
  "policy": {
    "inject_where": [{"dim": "trace_id", "op": "Eq", "values": ["<slice>"]}],
    "redact_dims": ["trace_id"],
    "allow_measures": ["duration_us"],
    "max_window_secs": 86400,
    "max_groups": 100,
    "focus": "read-only reporting for one slice"
  }
}
```

**Value allow-list** (may only see these `trace_id` values, injected if unconstrained):
```json
{"policy": {"allow_values": {"trace_id": ["a", "b"]}}}
```

**Column allow-list** (may reference ONLY these columns — anything else is a clean denial):
```json
{"policy": {"allow_dims": ["trace_id", "ts"], "allow_measures": ["duration_us"]}}
```

**Role + per-user pin** (author the SHAPE once as a role, mint many keys each pinned to one user's rows):
```json
// PUT /v1/admin/roles/tenant-viewer  → the reusable shape
{"policy": {"redact_dims": ["trace_id"], "max_window_secs": 604800}, "can_write": false, "admin": false}
// then per slice:  POST /v1/admin/keys  (pin a REAL column of this schema)
{"principal": "slice-key", "role": "tenant-viewer", "pin": {"trace_id": "<value>"}}
```

## Guardrails you must respect

- **Tenant blast radius**: you can only mint within YOUR tenant (`default`); the engine enforces this — a minted key can never reach another tenant's data.
- **`admin: true` is root-of-tenant** — a key you mint with `admin:true` can mint/revoke other keys. Grant it almost never; default `false`.
- **`can_write: true`** lets the key INGEST. Default `false` for reporting/analysis keys.
- **Fail-closed**: an unknown column in a policy, or a self-contradictory policy (a pin outside its own `allow_values`), is rejected at mint/enforce — you get an error, never a silently-broken grant.
- **Redact > allow_dims for PII**: to hide a column, `redact_dims` it (blocks every position incl. filters). Omitting it from `allow_dims` only blocks reference when `allow_dims` is non-empty AND misses a filter-tree edge if you forget one — redaction is the safe default for PII.

## Verify a policy — you MUST run queries (here's how)

Minting is half the job; verifying the scope actually holds is the other half — and that requires running queries **with the new key's token** (not your admin token). This admin doc doesn't restate the full query API; the authoritative per-key reference is **`GET https://oliverdb-trillo.tnt-trillo.svc.cluster.local:8080/onboard` fetched with the NEW key** — it returns that key's schema, its scope manifest, and copy-runnable SQL + DSL examples bounded to its scope. Always fetch it right after minting.

Minimal query call (native DSL) — `POST /v1/query` returns `policy_notes` showing what `enforce()` rewrote (pins, clamps, caps); the field is ABSENT/empty when the query needed no rewrite (an in-scope query), so its presence is the signal the boundary was applied:
```bash
curl -s https://oliverdb-trillo.tnt-trillo.svc.cluster.local:8080/v1/query -H "authorization: Bearer <NEW_KEY_TOKEN>" \
  -H 'content-type: application/json' \
  -d '{"agg":"Count","group_by":["trace_id"],"from_ms":0,"to_ms":4000}'
# or SQL:  -d '{"sql":"SELECT trace_id, count(*) FROM t GROUP BY trace_id"}'
```
`to_ms=4000` is just past this tenant's last row (see Data range) — pick a `to_ms` near the real data, NOT `i64::MAX`, or the window clamp can silently return 0 rows (false pass).

**Two-check verification recipe:** (1) a query that TRIES to exceed the scope (all values of a pinned dim, a redacted column, an over-wide window) must come back NARROWED (only the pinned slice, with the rewrite in `policy_notes`) or DENIED (HTTP 400 plaintext) — never leaking; (2) confirm `policy_notes` lists the pins/clamps you authored.

For adversarial assurance beyond spot-checks, `oliverdb-engine`'s `rbac_harness` binary machine-proves a policy scoped==unscoped-over-filtered-rows across 6000 generated pairs — run it in CI against any policy shape you standardize as a role.

## Limitations — what a Policy can NOT do (so you don't over-promise)

- **No recency floor.** `max_window_secs` caps window WIDTH, not age. "Only the last 7 days" is NOT expressible — a caller may query any 7-day-wide slice, however old. If you need rolling recency, enforce it OUTSIDE the policy (rotate the pinned window, or filter upstream).
- **Three overlapping value-scoping tools; pick deliberately.** To restrict WHICH rows/values a key sees you have: `inject_where` (hard RLS pin, conflict=deny — use for the ONE authoritative row scope, e.g. tenant), `allow_values` (a menu the caller may pick within, injected if unconstrained — use when the caller legitimately chooses among allowed values), and `allow_dims`/`redact_dims` (COLUMN visibility, not value). Rule of thumb: pin identity with `inject_where`, offer a menu with `allow_values`, hide columns with `redact_dims`.
- **`allow_measures` needs ≥2 numeric columns to bite**, and the schema's default measure (if any) is exempt — see the schema section.
- **Policy is per-KEY, not per-request.** A key's scope is fixed at mint; to change it, `PUT` the key's policy or revoke + re-mint. There's no per-query scope override.

_Author least-privilege. Redact PII. Pin the row scope. Cap the window and detail. Grant `admin`/`write` almost never. Know the limitations above — don't promise recency you can't enforce._
