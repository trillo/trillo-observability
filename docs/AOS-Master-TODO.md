# AOS — Master TODO

**Living list.** Consolidates everything surfaced during the AOS issues campaign:
what's fixed-and-awaiting-deploy, new follow-ups/enhancements we discovered while
fixing, decisions still owed, and the remaining backlog. Companion to
`AOS-Issues-Triage-Plan.md` (per-issue detail) — this is the at-a-glance tracker.
Last updated 2026-08-22.

---

## A. Deploy & coordinate now (all fixes are on `develop`, nothing deployed)

**`tcs-service` develop** — 6 security fixes stacked:
| Issue | Commit | What |
|---|---|---|
| AOS-33 P0 | ed16b9d | access-link privesc |
| AOS-43 P0 pt1 | 5e5f0f3 | credential-class `/data` HARD_BLOCKED |
| AOS-46 P1 | 64f9926, d547864 | ownership forgery + createOnly/owner immutable-on-update |
| AOS-34 P2 | 7228ac2 | strip hidden/listHidden from custom-select results |
| AOS-43 P0 pt2 | 382c049 | remove dead plaintext-token fallback (tokens hashed since Apr) |
| AOS-23 P1 | d10bfe2 | 404 soft-deleted files on serve/signed-url/metadata |

**`aos-py-execution` develop** — AOS-44 (3e327a2, pod de-privilege) + AOS-17 (0682351,
coded-agent crash). Needs pod rebuild + restart.
**`tcs-metadata` develop** — AOS-44 toolkit (b50904f). Installed into the pod build.
**`tcs-gcp` develop** — AOS-44 Vertex proxy retry (59c4001).
**`trillo-aos` develop** — AOS-44 proxy productionization + metering (d79c49f).

**Coordination tasks (Anil + team):**
- [ ] Deploy the `tcs-service` stack to internal → prod (closes AOS-33/43/46/34/23 live).
- [ ] **AOS-44 Phase 0 (DevOps, immediate risk cut):** drop `storage.objectViewer` from
      the exec-pod GSA (leave Vertex-only). gs:// is read by the Vertex service agent, not
      the pod GSA, so this is safe — verified.
- [ ] **AOS-44 team smoke on internal:** one agent turn (proxy generate + tool-calling) +
      a knowledge ingest (embeddings via `/embed`); watch for the `[LLM-METER]` log line
      and confirm **no ADC/metadata errors**.
- [ ] **AOS-44 Phase 3 (DevOps, after smoke — now unblocked):** remove the
      Workload-Identity GSA binding from the exec KSA + apply deny-by-default egress
      NetworkPolicy (allow only AOS + `storage.googleapis.com`; deny metadata/link-local/
      RFC1918). The pod's last ADC path (Claude-on-Vertex) was removed, so nothing needs
      metadata anymore.
- [ ] **AOS-17 retest:** re-run a codeful agent post-deploy; the `.code`→`.type` fix
      un-masks the real error — confirm no deeper every-turn failure was hiding behind it.

---

## B. Follow-ups & enhancements we surfaced (NEW — not in the original triage)

These came out of doing the fixes. None are blocking; each is scoped.

- [ ] **Token hashing → keyed blind-index (AOS-43 defense-in-depth).** Tokens are bare
      SHA-256 today; `MFA_OTP` is low-entropy (`userId:6-digit`) so a DB leak is
      brute-forceable within its 5-min TTL. Switch `hashToken` to
      `CryptoUtil.blindIndex` (HMAC-SHA256), at least for MFA_OTP.
- [ ] **Block client writes to ALL `platform:true`/system attributes on create
      (AOS-46 "step beyond").** Today only the ownership attr is force-corrected on create.
      Generalize to reject/ignore client-supplied platform attributes — needs cataloguing
      which platform attrs are legitimately client-set at create before enforcing.
- [ ] **Ownership transfer / reassignment capability (AOS-46 offboarding).** Generic CRUD
      now forbids owner changes entirely; legitimate reassignment (user leaves; "move all of
      X's records to Y"; hand a record to a teammate) needs its own sanctioned surface:
      admin-gated (or owner-initiated), **audited**, per-record + bulk. Model after the
      `/share/*` surface (dedicated controller, not `/data`). → fold into a small plan.
- [ ] **`deleteFile` soft-delete recoverability (AOS-23).** `deleteFile` calls
      `storage.delete()` for BOTH soft and permanent delete, so soft-deleted GCS bytes are
      gone despite the "recoverable" API-doc promise. Gate `storage.delete()` on
      `permanent==true`.
- [ ] **AOS-44 metering depth.** The `[LLM-METER]` hook logs tenant/appId/model/op; add
      token & cost from the response `usageMetadata`, keyed on tenant (natural quota point).
- [ ] **AOS-44 streaming latency.** Only relevant if a future path adopts
      `generate_content_stream`; today the loop is non-streaming so the AOS hop adds no
      first-token latency. Revisit then (fallback = broker-pod for streaming).

---

## C. Decisions owed / tabled

- [ ] **AOS-10 P1 (class `acl` not enforced on `/data`) — TABLED, revisit.** Fix is known
      (remove the `!isAgentDelegation()` bypass at `AclAccessGuard.java:93`; keep the
      `isInternalCaller` bypass at :96). **Blocked on a decision:** platform-admin
      handling — (a) no bypass (secure-default, risks Data Manager on acl'd classes that
      omit `admin`), (b) admin bypasses acl on the direct path (lean; keeps Data Manager),
      (c) flag-gated `adminBypassesAcl`. Plus optional shadow-mode rollout beside the
      `AclAccessGuard.enabled` kill switch.
- [ ] **Shared `isInternalCaller` trust-boundary question (AOS-10 + AOS-11).** Both guards
      bypass for requests with no `X-Forwarded-By` (direct-to-AOS). That's relied on by
      function `ctx.*` callbacks today; whether to tighten it is a separate cross-cutting
      call.

---

## D. Remaining backlog (open in the triage plan — Phases 2-6)

Detail + severities live in `AOS-Issues-Triage-Plan.md`. Summary of what's left:

- **Phase 2 — Agent runtime:** AOS-02b (one-shot exec unimplemented), AOS-14 (empty
  function `parameters` in discovery), AOS-18 (turns >25s → gateway 504). *(AOS-17 done.)*
- **Phase 3 — Telemetry ingestion endpoint:** build per `Telemetry-Ingestion-Endpoint-Design.md`.
- **Phase 4 — Data correctness & migration:** AOS-16, 36, 35, 21, 22, 31, 24, 39.
- **Phase 5 — Multi-tenancy:** AOS-15, 37.
- **Phase 6 — Error-shape / DX / authoring:** AOS-07 P-06/P-07, AOS-08, 09, 25, 26, 27,
  28, 29, 30, 38, + inline (guest-secrets, `authGuestEnabled`) — both need repro docs.

**Phase 1 — Security: COMPLETE** except AOS-10 (tabled, §C). Fixed: AOS-33, 43(pt1+pt2),
44(code), 11(already-fixed), 46, 34, 23. Verified-already-implemented (stale reports):
AOS-11, AOS-43-pt2 hashing, AOS-45.

---

## E. Other workstreams (parked, for continuity)

- **Trillo Agent Observability as SaaS (~Q1 2027).** Website rework then: dual deployment
  model (Managed SaaS vs BYOC) + self-serve CTA; keep BYOC copy "additively true" now.
  See `trillo-ai-website-draft` README.
- **Website draft** (`trillo-ai-website-draft`): homepage, AOS platform page, catalog, 3
  product pages done. Next: Company/About; per-product SaaS deployment section later.
