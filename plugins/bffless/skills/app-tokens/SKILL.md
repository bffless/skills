---
name: app-tokens
description: Mint, use and revoke BFFless app tokens — project-bound bearer credentials with scopes for agents, MCP connectors, headless browsers and CI — and exchange one for a cookie session
---

# App Tokens

An **app token** (`bfat_…`) is a bearer credential bound to **one project** and delegated a
set of **scopes**. It is what an MCP connector holds after OAuth consent, and what you mint
by hand when an agent, headless browser or CI job should act as a member on one project
only. Needs CE ≥ 0.4.43; `neverExpires` + paged list ≥ 0.4.51; session exchange ≥ 0.4.50.

## Which credential

| Credential | Header | Bound to | `requiredScopes` checked | Mint |
| --- | --- | --- | --- | --- |
| Session | cookies | the person | no | sign in |
| API key | `X-API-Key` | project or global | no | Settings → API Keys / `create_api_key` |
| **App token** | `Authorization: Bearer bfat_…` | **one project** | **yes** | Settings → App Tokens / `POST /api/app-tokens` / OAuth consent |

- Effective permission = the member's own permission on that project ∩ the token's scopes.
  It never elevates; a global admin's token is still fenced to its project.
- Precedence when several are sent: `X-API-Key` → `bfat_` bearer → session cookie →
  custom-domain cookie. A non-`bfat_` Bearer is ignored.
- A bearer request is an **API request**: failures are JSON `401`/`403`, never a `302`.
- There is **no MCP tool** for app tokens on the admin server; use the REST endpoint with a
  session, or the admin UI.

## Mint

Session-only (`POST /api/app-tokens` refuses API keys and tokens: a credential cannot mint
a credential). Any member ≥ viewer on the project.

```bash
curl -s -X POST https://admin.<host>/api/app-tokens -H "Content-Type: application/json" -b "sAccessToken=…" -d '{"name":"Claude — workflow","project":"owner/repo","scopes":["workflow:read","workflow:run"],"neverExpires":true}'
```

| Field | Rules |
| --- | --- |
| `name` | 1–255 chars |
| `project` | `owner/repo` |
| `scopes` | 1–20, each `namespace:verb` lowercase (`/^[a-z][a-z0-9_-]*:[a-z][a-z0-9_-]*$/`). Your app's vocabulary — nothing is registered. `auth:` is CE's reserved namespace |
| `expiresAt` | ISO-8601; default 90 days, max 365 days ahead |
| `neverExpires` | `true` → `expiresAt: null`. Mutually exclusive with `expiresAt` (400) |

The secret is in the create response **once**; CE keeps a SHA-256 hash. `lastUsedAt`
updates at most once a minute.

## List / revoke

```
GET    /api/app-tokens?includeInactive=true&limit=200&cursor=<nextCursor>
DELETE /api/app-tokens/:id
```

A bare `GET` is the **first 50 active** tokens only (`{ data, nextCursor }`); revoked and
expired ones need `includeInactive=true`. `limit` 1–200; follow `nextCursor` until `null`.
Scripts that used to iterate "all tokens" from a bare GET must page.

## Use against a pipeline

```bash
curl -s https://<project-host>/api/mcp -H "Authorization: Bearer bfat_…" -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

| Outcome | Status | Body |
| --- | --- | --- |
| missing a scope in the rule's `auth_required.requiredScopes` | 403 | `insufficient_scope: missing <scope>` + `WWW-Authenticate: Bearer error="insufficient_scope"` |
| host resolves to a different project | 403 | `{ code: "TOKEN_PROJECT_MISMATCH" }` — checked **before** visibility (CE ≥ 0.4.58): public deployments and `bypassVisibility` rules too |
| unknown / expired / revoked | 401 | JSON + `WWW-Authenticate: Bearer resource_metadata="…"` on domain hosts |

In expressions and `function_handler`: `user.credential == "app_token"`, `user.scopes`
(array), plus the member's `user.id` / `user.role` / `user.projectRole`. See **pipelines**.

## Exchange for a session

For a client that holds only a token but needs cookie-authenticated pages (a headless
browser driving the admin UI, CI fetching a private deployment):

```bash
curl -s -X POST https://admin.<host>/api/auth/session/from-app-token -H "Authorization: Bearer bfat_…" -c cookies.txt
```

- No body. Token must carry **`auth:session`** — include it in `scopes` at mint time.
- Response = `POST /api/auth/signin`; cookies set (valid on subdomains via `COOKIE_DOMAIN`).
- `GET /api/auth/session` then reports `session.via: "app_token"` (+ `appTokenId`).
- Session lifetime is SuperTokens' normal one: **not** clipped to the token's expiry, and
  revoking the token does **not** end sessions already minted.

| Status | `code` | When |
| --- | --- | --- |
| 401 | `unauthorized` | no `bfat_` bearer; token unknown/expired/revoked; user disabled; or `REQUIRE_PROJECT_MEMBERSHIP` on and no role |
| 403 | `insufficient_scope` | lacks `auth:session` (`missingScopes` names it) |
| 403 | `token_project_mismatch` | host resolves to another project |
| 409 | `user_not_exchangeable` | legacy user row unknown to SuperTokens |

## OAuth-issued tokens

An OAuth access token from CE's built-in authorization server (claude.ai connector, Claude
Code `/mcp`) **is** an app token: `kind: oauth`, 1 h, refreshed by a rotating `bfrt_`
refresh token (30 d; reuse revokes the family). It appears under Settings → App Tokens and
is revoked the same way. Server side: see **mcp**.

## Gotchas

1. **Don't reach for `allowApiKey`** to admit a token — it is neither needed nor enforced.
2. **One project per token.** For two projects mint two tokens; a scoped alias also needs
   the member to hold a project role there.
3. **`requiredScopes` never blocks a session or API key.** If a scope check "doesn't work"
   in the browser, that is why — test with the bearer.
4. **Legacy sync default changed** (0.4.51): bare `GET /api/app-tokens` no longer returns
   revoked/expired tokens.
