# UI-team handoff: login-first OAuth completion for AOS app plugins

**Audience:** the UI team building the AOS app SPA (starting with the Agent_Observability app console).
**Why:** the SRE Copilot plugin (and any future AOS-app plugin) signs in with
OAuth. AOS handles the protocol, but the **app's own login page must complete the
flow** — otherwise auth hangs (the terminal shows "auth pending" forever). This
doc is the exact contract the login page must implement.

Server side: `OAuthController` (tcs-service). Client side: your SPA.

---

## 1. Where the browser lands (what changed server-side)

For an **app-bound** OAuth client (e.g. `sre-claude-code`), AOS resolves the login
page from that app's **`AppConfig.frontendUrl`** (per-app, per-env) and 302-redirects
the browser to:

```
<AppConfig.frontendUrl>/login?oauth_state=<opaque-state>
```

- The path is **`/login`** (server constant `OAUTH_LOGIN_PATH`). Your SPA must
  handle the `oauth_state` query param on that route. (If you need a different
  route, tell us — it's one constant.)
- If `frontendUrl` isn't configured, AOS errors the flow out (no fallback) — the
  browser never reaches you. So set `AppConfig.frontendUrl` for the app.

## 2. The completion handshake — what the login page must do

```
Browser lands: <frontendUrl>/login?oauth_state=STATE
        │
        ├─ 1. Detect oauth_state  → enter "OAuth completion" mode
        ├─ 2. Authenticate (login, or reuse existing session) — KEEP oauth_state
        ├─ 3. Select / confirm tenant (if multi-tenant)
        ├─ 4. POST {AOS}/api/v2.0/oauth/complete   (Bearer token + state)
        ├─ 5. Response → { data: { redirectUrl: "http://localhost:PORT/callback?code=…&state=…" } }
        └─ 6. Verify redirectUrl is loopback, then window.location = redirectUrl
                        │
                        └─ Claude Code's local listener catches the code → done
```

### Step detail

1. **Detect** — on load, read `oauth_state`. Its presence means "finish an OAuth
   authorization," not a normal app session.
2. **Authenticate** — if no session, show login; if a session exists, reuse it.
   **Preserve `oauth_state` across the whole login round-trip** (query, app state,
   wherever) — *losing it here is the #1 failure mode.*
3. **Tenant** — if the app is multi-tenant, let the user select/confirm the tenant
   so the bearer token carries it.
4. **Complete** — call:
   ```
   POST  {AOS_API_BASE}/api/v2.0/oauth/complete
   Authorization: Bearer <the user's token>
   Content-Type: application/json
   Body: { "state": "<oauth_state>" }
   ```
   - `AOS_API_BASE` **must be the same AOS host that issued `/authorize`** — the one
     the plugin's MCP points at (e.g. `https://aos-dev.trillo.ai`). The flow session
     lives in that AOS's DB, so completing against a different host/env fails.
   - (A form-encoded variant `POST /oauth/complete` with `state=…` is also accepted.)
5. **Response** — `200` with:
   ```json
   { "data": { "redirectUrl": "http://localhost:8090/callback?code=…&state=…" } }
   ```
6. **Redirect** — **verify `redirectUrl` is a loopback URL** (`http://localhost |
   127.0.0.1 | [::1]`), then `window.location.assign(redirectUrl)`. That hands the
   code to Claude Code's local listener and the terminal flips to authenticated.

## 3. Errors to handle

- **Flow session expired / not found** (5-min TTL) or `/complete` 4xx → show
  "authorization expired or invalid, please retry from the terminal"; do **not**
  redirect.
- **`/complete` 401** → token invalid/expired → re-authenticate, keep `oauth_state`.
- **User cancels** → return to the app normally; the terminal listener times out on
  its own.

## 4. Gotchas / must-dos

- **Don't drop `oauth_state`** through login — the single most common bug.
- **Same-origin/env for `/complete`** — target the AOS host that issued `/authorize`.
- **CORS** — `/complete` is cross-origin (your SPA → AOS). `AppConfig.frontendUrl`
  is auto-included in the AOS CORS allowlist, so if `frontendUrl` is set correctly
  this is already handled; if you serve the SPA from another origin, add it to
  `AppConfig.allowedCorsOrigins`.
- **No consent screen** — these are first-party clients (auto-consent). You do
  **not** render a consent page; you just complete.
- **Loopback-only navigation** — the server only ever returns a loopback
  `redirectUrl`, but double-check client-side before navigating (open-redirect
  safety).

## 5. Test checklist

- [ ] `/login?oauth_state=…` with **no** session → login → completes → browser
      lands on `localhost:<port>/callback?code=…` → terminal authenticated.
- [ ] `/login?oauth_state=…` with an **existing** session → completes without a
      second login (oauth_state not dropped).
- [ ] Multi-tenant app → tenant selection happens before `/complete`.
- [ ] Expired flow session → clean error, no redirect.
- [ ] `/complete` targets the correct AOS host (check Network tab: one POST to
      `…/api/v2.0/oauth/complete`, 200, with a loopback `redirectUrl`).

## References

- `OAuthController` (tcs-service) — `/authorize`, `/complete`, `/token`.
- `SRE-Plugin-Internal-Setup.md` — client registration, `SRE_MCP_URL`, invalid_client.
- Plan-59 — login-first OAuth.
