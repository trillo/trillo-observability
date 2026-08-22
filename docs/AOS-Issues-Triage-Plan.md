# AOS Issues — Triage & Work Plan

**Sources:** AOS-05 (skills docs, team retest 2026-08-16), AOS-06 (platform issues,
retest 2026-08-16), AOS-07 (plugin issues, retest 2026-08-16), plus two inline
items (guest-secrets, authGuestEnabled). **Internal.**

## Numbering — read this first

The **AOS-06 platform doc's internal `AOS-NN` are the canonical granular IDs.**
They collide with the top-level list first circulated:

| First-list ID | = canonical | Item |
| :-- | :-- | :-- |
| AOS-08 | **AOS-10** | class ACL not enforced on `/data` |
| AOS-09 | **AOS-11** | function `allowedAppRoles` not enforced on `/fn` |
| AOS-10 | **AOS-32** | no admin access to user-owned folders (Dropbox) |
| AOS-11 | *(none)* | guest token can't read integration secrets — **no detail doc** |
| AOS-12 | *(none)* | `authGuestEnabled` update≠fetch — **no detail doc** |

## Status snapshot

- **Open:** 34 platform (AOS-06) + 2 plugin (AOS-07) + 2 inline = **~38**.
  Platform severity: **2 P0, 15 P1, 8 P2, 8 P3** (AOS-40 held/monitor, not counted).
- **Fixed / closed (do not rework):** AOS-41 (P0 mint-ownership) + AOS-42 (P1 raw-SQL
  bypass) — *fixed by us, the SEC-01/02 work*; AOS-19 (empty list_classes), AOS-20
  (run_agent name guessing); AOS-05 all 8 skill-doc issues (v0.4.0); AOS-07 P-08 +
  packaging.
- **Tracked separately:** AOS-44 (SSRF → GCP service-account token) — critical, its
  own doc (`aos-44-pod-deprivilege-design.md`). **Phase 1+2 code done + pushed to
  `develop` 2026-08-22** (aos-py-execution 3e327a2, tcs-metadata b50904f, tcs-gcp
  59c4001, trillo-aos d79c49f): pod model calls now proxy through AOS (Gemini
  generate + embeddings), pod's ADC/Claude-on-Vertex path removed, retry moved
  AOS-side, per-call LLM timeout, `[LLM-METER]` hook. **Remaining:** team smoke on
  internal + Phase 0 (drop pod-GSA storage role) + Phase 3 (remove GSA binding +
  egress NetworkPolicy — now unblocked). Needs deploy.

---

## Phase 1 — Security (credibility-critical; one coherent surface)

The generic-REST + auth surface, in `tcs-service` / `tcs-core`. Start here.

- **AOS-33 · P0** — access-link privesc: non-admin mints link for `admin`, exchanges with **no auth** → admin session. **✅ FIX in `tcs-service` develop `ed16b9d` (needs deploy).** Root cause: the target guard checked the bare user row while the exchange minted the tenant/AppRole-resolved identity → app-admins (admin via appRole) passed; and the endpoint only required auth. Fix: (a) reject targets privileged in any active tenant by resolved role/appRoles; (b) require caller be admin or a privileged function-context. *(access-link controller/service, plan-76)*
- **AOS-43 · P0** — generic `/data` blocklist covered only `AppSecret`/`OtpVerification`; `UserToToken` etc. readable, tokens **plaintext** → refresh into admin. **✅ Part 1 FIX in `tcs-service` develop `5e5f0f3` (needs deploy):** unconditional `HARD_BLOCKED` set in `PlatformClassGuard` (credential/token/PII classes denied in all modes + for function context; dedicated controllers bypass the generic path, so legit function access is unaffected). **Part 2 open:** hash `UserToToken` tokens (currently plaintext) — separate follow-up. *(generic CRUD blocklist done; token storage pending)*
- **AOS-10 · P1 — ⏸ TABLED 2026-08-22 (investigated; revisit).** Confirmed real. Root
  cause: `AclAccessGuard.isPermitted` (called by `DataController`/`QueryController` on
  every op) short-circuits at `AclAccessGuard.java:93` — `if (!ctx.isAgentDelegation())
  return true;` — so class `acl` is enforced only on agent-delegation tokens; direct
  user/UI JWTs skip `AclGate.classPermits` entirely. Fix = remove that bypass (keep the
  `isInternalCaller` bypass at :96 — fn `ctx.data.*` callbacks rely on it). Blast radius
  bounded: apps without acl stay open (plan-79 empty=open); secret/system classes stay
  covered by `PlatformClassGuard`; `AppRoleResolver` injects coarse `admin`/`guest` but
  **no coarse `user`**. **Open decision before coding:** platform-admin handling —
  (a) no bypass (secure-default, risks Data Manager on acl'd classes omitting `admin`),
  (b) admin bypasses acl on direct path (lean; keeps Data Manager), (c) flag-gated
  `adminBypassesAcl`. Plus optional shadow-mode rollout (log-only) beside the existing
  `AclAccessGuard.enabled` kill switch.
- **AOS-11 · P1 — ✅ ALREADY ENFORCED (close; no code).** `/fn` sync+async + MCP
  `tools/call` all run `ExecAuthzGuard.enforce` (role rank + `allowedAppRoles`) via
  `FnService.lookupFunctionCode:239`, wired since 2026-07-09/10 — a month before the
  2026-08-16 report. The report was against a stale build (deploy lag). Only residual is
  the shared `isInternalCaller` bypass (`ExecAuthzGuard.java:76`), same parked
  trust-boundary question as AOS-10:96. Closes on deploy.
- **AOS-46 · P1 — ✅ FIXED (in `tcs-service`, needs deploy; build-verified, uncommitted→pushed).**
  Two gaps in `DataServiceImpl`: (1) `validateAndSanitize` stripped `computed`+`readOnly`
  but **never `createOnly`** — a client could mutate any create-only field on `/data`
  update, incl. reassigning `idOfUser` (declared `createOnly`+`representsOwnership`) to
  another user; (2) create-time forgery — `DataUtil.ensureHasOwner` fills owner only
  *if absent*, so a client POSTing `{idOfUser:<victim>}` on create had it honored.
  Fix: (1) on update strip `createOnly` AND the ownership attribute
  (`representsOwnership`) — so **the owner is immutable via generic `/data` update for
  everyone, admins included**, and keying on `representsOwnership` (not only createOnly)
  also protects custom classes whose owner attr isn't marked createOnly; (2) new
  `forceOwnerToCaller` helper on the public `create`/`upsert` methods overwrites the
  ownership attr with the caller (exempt: `internal*` raised-privilege paths via
  skipValidation, and admin on `adminBypassesOwnership:true` classes for
  create-on-behalf). `readOnly` left create-settable by design.
  **Follow-ups:**
  - *(Anil's "step beyond", needs analysis)* block client writes to ALL
    `platform:true`/system attributes on create too — requires cataloguing which platform
    attrs are legitimately client-set at create before enforcing; dedicated pass.
  - *(ENHANCEMENT — new capability)* **Ownership transfer / reassignment.** Generic CRUD
    now forbids owner changes entirely, so legitimate reassignment (user offboarding,
    "move all of X's records to Y", handing a record to a teammate) needs its own
    sanctioned surface: admin-gated (or owner-initiated), audited, and per-record + bulk
    "reassign everything owned by user X". Model it after the `/share/*` surface (a
    dedicated controller, not `/data`). Fold into a small plan when prioritized.
- **AOS-34 · P2** — `list_users` returns bcrypt hashes. *(strip `password` from projection)*
- **AOS-23 · P1** — soft-deleted files still served + fresh signed URLs. *(FileController: 404 + refuse download-url on deleted)*
- **AOS-45 · P2** — no max page size on `/data`. *(clamp `end` server-side)*

## Phase 2 — Agent runtime (gates the observability AI + demo)

Our observability products run on agents; right now codeful agents are 100% down.

- **AOS-17 · P1** — codeful agents crash every turn (`_AgentStateAPI`→`ErrorDetail.code`). *(aos-py-execution agent_handler)* — **lead here.**
- **AOS-02b · P1** — one-shot agent execution unimplemented (`ONESHOT_PROGRAMMATIC_PENDING`); only the `ctx.llm.process_document` workaround runs. *(pod dispatch route)*
- **AOS-14 · P2** — `list_functions`/`describe_function` return empty `parameters`. *(discovery projection)*
- **AOS-40 · P2 (monitor)** — agents do speculative writes from read-only questions; no read-only/confirm tool model.
- **AOS-18 · P3→P1-in-context** — turns >25s return gateway 504 not the running handle. *(gateway timeout / cap waitSeconds)*

## Phase 3 — Telemetry Ingestion Endpoint (build)

Per `Telemetry-Ingestion-Endpoint-Design.md`. Builds on Phase-1 hardened auth. Demo-useful. Pull ahead of Phase 2 if the demo needs it sooner.

## Phase 4 — Data correctness & migration

- **AOS-16 · P1** — `readOnly` attrs silently dropped on update → **data loss** (computed invoice totals never stored). *(generic update honors readOnly for server callers, or fail loud)*
- **AOS-36 · P1** — in-place column ALTER (type change, `unique`) silently no-ops while deploy reports success. *(trillo-aos migration)*
- **AOS-35 · P1** — failed migration → whole-app data API `412` outage; raw SQL error surfaced. *(scope failure to the entity; pre-check NOT NULL)*
- **AOS-21 · P1** — inline `save_content` leaves file `deleted:true` → invisible. *(finalize semantics for inline path)*
- **AOS-22 · P1** — `get_content(version=N)` ignores version → history unreachable.
- **AOS-31 · P1** — `ctx.files.list(folderId)` ignores folder → returns all files + platform-internal. *(honor folderId; scope sourceClass)*
- **AOS-24 · P2** — `upload_succeeded` accepts with no bytes in GCS → phantom file.
- **AOS-39 · P1** — read-validate-write race → oversell; no atomic/lock/version primitive. *(offer atomic conditional update; fix the Functions skill's recommended pattern)*

## Phase 5 — Multi-tenancy (only if we claim it)

- **AOS-15 · P1** — `miscInfo.multiTenant:true` not carried to `AppConfig.multiTenant`; redeploy resets it; not editable from plugin. *(DeployAppMetadata.bootstrapAppConfig — the known boolean/bootstrap gotcha; I have this)*
- **AOS-37 · P1** — reachable multi-tenant state does not isolate tenant reads. *(tenant scoping in generic queries)*

## Phase 6 — Error-shape / DX / authoring polish

- **AOS-07 P-06 · P2** — `md_update` merges instead of replacing → can't remove a field; stale keys accrue. *(metadata update semantics)*
- **AOS-07 P-07 · P3** — `aos_call` doesn't send `x-app-id` header (workaround `?appId=`); doubled/malformed error string. *(authoring MCP tool)*
- **AOS-08 · P3** — raw REST `/agent/{name}/invoke` drops the user message (`content:null`).
- **AOS-09 · P2** — task/conversation event routes: wrong documented URL 500s, `/conversations/{id}/events` returns empty, 401 with success-shaped body.
- **AOS-25 · P2** — folder-name uniqueness is tenant-wide but visibility per-owner → existence oracle + name block.
- **AOS-26 · P3** — `ctx.files` has no folder read/list/rename/move (REST get/delete exist, unsurfaced).
- **AOS-27 · P3** — `ctx.files.list` typed `list` but returns a page dict; docstring example raises.
- **AOS-28 · P3** — `ctx.files.delete` returns a zeroed stub.
- **AOS-29 · P3** — literal `null` in GCS object paths for nested folders.
- **AOS-30 · P3** — unsupported HTTP method returns 500 not 405.
- **AOS-38 · P3** — AppConfig password-policy fields ignored; hard-coded policy enforced.
- **Inline · needs detail** — guest token can't read integration secrets; `authGuestEnabled` update≠fetch (both need repro docs before triage).

---

## Why this order
Security first — two live account-takeovers, and it's the core product claim. Agents
second — they gate the observability products and the demo. Telemetry endpoint third
(ready to build, wants the hardened auth). Correctness/migration/MT are serious but
not actively exploited or demo-blocking. DX polish last.

## Cross-cutting notes
- **Phase 1 is one investigation, many fixes** — `/data`, `/fn`, access-link, and the
  admin/user paths share the authorization layer the agent-delegation path already
  applies correctly; the theme is "REST bypasses the gate."
- **We've done this class before** (AOS-41/42 fixed). Same repos, checked out here.
- Keep exploit repro in the source security docs, not restated here.
