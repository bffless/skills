---
name: proxy-rules
description: Forward requests to backend APIs without CORS
---

# Proxy Rules

**Docs**: https://docs.bffless.app/features/proxy-rules/

Proxy rules forward requests from your static site to backend APIs, eliminating CORS issues and hiding API endpoints from clients.

## Configuration Levels

**Project defaults**: Apply to all aliases unless overridden

**Alias overrides**: Specific aliases can have their own proxy rule sets assigned

## Rule Sets

Proxy rules are organized into **rule sets** — named, reusable groups of rules.

An alias can have **multiple rule sets** attached. When multiple rule sets are assigned, their rules are merged in priority order (first set wins if two sets define the same path+method).

This allows logical grouping, e.g.:
- `api-proxy` — routes to your backend API
- `pipelines` — pipeline-based handlers for chat, forms, etc.
- `auth-proxy` — cookie-to-bearer token transformation

### Assigning Rule Sets to Aliases

Use `update_alias` with `proxyRuleSetIds` (array) to attach one or more rule sets:

```
update_alias(
  repository: "owner/repo",
  alias: "production",
  proxyRuleSetIds: ["rule-set-id-1", "rule-set-id-2"]
)
```

The legacy `proxyRuleSetId` (singular) still works for backwards compatibility but only supports one rule set.

**Important**: After creating proxy rules, you must assign the rule set(s) to an alias for rules to take effect. Rules won't be active until linked to an alias.

## Rule Structure

Each rule specifies:
- **Path pattern**: Which requests to intercept
- **Target**: Backend URL to forward to
- **Strip prefix**: Whether to remove the matched prefix

## Pattern Types

**Prefix match** (most common):
```
/api/* → https://api.example.com
```
Request to `/api/users` forwards to `https://api.example.com/users`

**Exact match**:
```
/health → https://backend.example.com/status
```
Only exact path matches, no wildcards

**Suffix match**:
```
*.json → https://data.example.com
```
Matches any path ending in `.json`

## Strip Prefix Behavior

With strip prefix ON (default):
```
/api/* → https://backend.com
/api/users → https://backend.com/users
```

With strip prefix OFF:
```
/api/* → https://backend.com
/api/users → https://backend.com/api/users
```

## Common Patterns

**API proxy**: `/api/*` → your backend server

**Third-party API**: `/stripe/*` → `https://api.stripe.com` (hides API from client)

**Microservices**: Different prefixes route to different services

## Security Notes

- All proxy targets must use HTTPS
- BFFless validates targets to prevent SSRF attacks
- Headers like `Host` are rewritten to match target
- Client IP forwarded via `X-Forwarded-For`

## Troubleshooting

**Requests not proxying?**
- Check path pattern matches request URL exactly
- Verify rule set is assigned to the alias via `update_alias`
- Confirm target URL is HTTPS and reachable

**Getting CORS errors still?**
- Ensure the proxy rule set is assigned to the alias being accessed
- Check browser DevTools for actual request URL (may not be hitting proxy)
