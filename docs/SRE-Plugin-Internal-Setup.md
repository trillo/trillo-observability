# SRE Copilot Plugin — Internal Setup & Troubleshooting

**Audience:** Trillo internal team (deploy / support). **Not for customers** — the
customer-facing guide is `sre-claude-plugin/docs/user-guide.md` in the public repo.

Covers: how the plugin authenticates, where OAuth clients are registered, how to
point the plugin at a dev server, and how to diagnose the common auth errors.

---

## 1. How auth works (one paragraph)

The `sre` MCP server in `sre-claude-plugin/.mcp.json` is an HTTP MCP server with
**OAuth** (`clientId: sre-claude-code`, `callbackPort: 8090`). Claude Code runs the
Authorization Code + PKCE flow against **AOS** (served by **tcs-service**). The
client is a **native public / loopback** client (RFC 8252): no secret, and the
redirect is a `http://localhost:<port>/…` loopback that Claude Code stands up at
runtime. It is **first-party**, so consent is auto-approved (no consent screen).

## 2. Where OAuth clients are registered — **in code, not a UI**

Clients live in a hardcoded map in:

```
tcs-service/src/main/java/io/trillo/service/service/OAuthClientRegistry.java
```

There is **no admin UI and no DB table** for this yet (a `schema0.oauth_client_tbl`
+ CRUD surface is a documented future item). To add or change a client, edit the
`CLIENTS` map and redeploy tcs-service.

Registered clients:

| client_id | display name | redirect model | consent | app binding (appName) |
| :-- | :-- | :-- | :-- | :-- |
| `trillo-claude-code` | Claude Code | native loopback (any localhost port) | first-party / auto | — (Designer platform login) |
| `sre-claude-code` | Trillo SRE Copilot | native loopback (any localhost port) | first-party / auto | `Agent_Observability` |

**Key rule:** for a `nativeLoopback` client the **callback port does not matter** —
any `http://localhost|127.0.0.1|::1` redirect on *any* port passes validation
(`isLoopbackRedirect`). So `callbackPort` in `.mcp.json` (8090 for SRE, 8080 for
authoring) is cosmetic; it never causes an `invalid_client`.

### Adding a client

```java
private static final Map<String, Client> CLIENTS = Map.of(
    "trillo-claude-code",
    new Client("trillo-claude-code", "Claude Code", List.of(), true, true, null),
    "sre-claude-code",
    new Client("sre-claude-code", "Trillo SRE Copilot", List.of(), true, true,
        "Agent_Observability"));
```

`Client(clientId, displayName, redirectUris, nativeLoopback, firstParty, appName)`.
Use `redirectUris = List.of()` + `nativeLoopback = true` for CLI/desktop clients.
The last arg is the **appName** the client authenticates for (`null` = unbound,
platform login). Bind the **appName** (stable across envs), never an appId (differs
per env — see §3). Future SRE apps are just more clients:
`Neoclouds_Observability`, `Data_Center_Observability`.

## 3. Login page resolution (app-bound clients)

An app-bound client's login page is the target app's own SPA, resolved at
`/authorize` from that app's `AppConfig.frontendUrl`:

```
clientId → (registry) appName → hosted_app_tbl → appId/schema (per env)
        → AppConfig.frontendUrl → <frontendUrl>/login?oauth_state=…
```

- The **appName** is the stable binding; the **appId** is resolved dynamically per
  environment via `hosted_app_tbl` (same pattern as `OAuthConfigService`), so one
  code constant works in every env.
- **You must set `AppConfig.frontendUrl`** for the app (`Agent_Observability`) in
  each environment. If it's blank or the app isn't in `hosted_app_tbl` there,
  `/authorize` returns `server_error` (no fallback) — deliberately, so a user is
  never sent to the wrong login.
- An optional **`app_name`** query param on `/authorize` overrides the registry
  default — the forward hook for letting Claude Code select among
  `Agent_Observability` / `Neoclouds_Observability` / `Data_Center_Observability`.
  Not wired into discovery yet; the registry default is what's used today.
- The unbound authoring client (`trillo-claude-code`) ignores all of this and uses
  the platform login (`tcs.oauth.login.url` / `tcs.app.server.url`).

## 4. Install & point the plugin at a server

Install the marketplace and plugin (both are named `sre`; the marketplace comes
from the `sre-claude-plugin` repo — the old `tao-claude-plugin` name still
redirects):

```bash
# 1. add the marketplace (from the renamed repo)
claude plugin marketplace add trillo/sre-claude-plugin

# 2. install the plugin — plugin@marketplace, both "sre"
claude plugin install sre@sre
```

The MCP URL is env-overridable: `${SRE_MCP_URL:-https://aos.trillo.ai/api/v2.0/mcp}`
(defaults to production `https://aos.trillo.ai/api/v2.0/mcp`; override with `SRE_MCP_URL` for dev or self-hosted environments). For dev:

```bash
# 3. point at the dev AOS
export SRE_MCP_URL="https://aos-dev.trillo.ai/api/v2.0/mcp"
claude
```

Then `/mcp` → **sre** → authenticate. (Or set `SRE_MCP_URL` in Claude Code's
settings `env` block instead of the shell.)

**Critical:** the client is registered **per server**. `sre-claude-code` must be
deployed (via the registry above) on the **same** AOS/tcs-service instance that
`SRE_MCP_URL` points at. Authoring's `api.trillo.ai` and any dev host are separate
deployments with separate registries.

## 5. Deploy after a registry change

1. Edit `OAuthClientRegistry.java` (§2).
2. Build + deploy **tcs-service** to the target environment (dev first, then prod).
3. Verify the new client is live before customers try to authenticate.

## 6. Troubleshooting

### `invalid_client` — "client_id and redirect_uri do not match a registered client"
Almost always **the client_id isn't registered on the server the URL points at.**
Check, in order:
1. **client_id registered on THIS server?** — is `sre-claude-code` in the deployed
   `OAuthClientRegistry` on the host `SRE_MCP_URL` resolves to? (Deploy lag is the
   usual culprit: code merged but that env not redeployed.)
2. **URL vs registry host match?** — pointing at prod while the client is only on
   dev (or vice versa) produces this exact error.
3. redirect_uri is **not** the cause for a loopback client — any localhost port is
   accepted, so don't chase the callback port.

### `unauthorized` on a tool call
Session expired or never completed. Re-run `/mcp` → **sre** → authenticate.

### Connection not listed / won't start
Confirm the plugin is installed (`claude plugin install sre@sre`) and, for a
non-default host, that `SRE_MCP_URL` is exported in the same shell that launched
Claude Code.

### A "server error" that's really a malformed `SRE_MCP_URL`
A bad endpoint value often *looks* like a server-side failure but is a client-side
URL problem. Watch for a **glued word** (`…mcpclaude`, from `export SRE_MCP_URL=…mcp` immediately
followed by `claude` with no newline) or a **trailing flag** accidentally appended
to the value. Note a **`401` does not validate the path** — the challenge fires
before routing, so a 401 can come back even when the path is wrong. Fix: print the
value (`echo "$SRE_MCP_URL"`) and confirm it is exactly
`https://<host>/api/v2.0/mcp` — no trailing characters, spaces, or joined words —
then re-`export` and relaunch.

## References

- `OAuthClientRegistry.java` (tcs-service) — the client list + loopback rule.
- Plan-59 — Claude Code authoring via OAuth+MCP (the flow this reuses).
- `sre-claude-plugin/.mcp.json` — the SRE client config.
- `Enterprise_AI_Agent_Observability_SRE_Copilot_Plugin_Design.md` — the plugin design.
