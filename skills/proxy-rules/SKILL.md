---
name: proxy-rules
description: Forward requests to backend APIs without CORS
---

# Proxy Rules

**Docs**: https://docs.bffless.app/features/proxy-rules/

Proxy rules forward requests from your static site to backend APIs, eliminating CORS issues and hiding API endpoints from clients.

## Configuration Levels

**Project defaults**: Apply to all aliases unless overridden

**Alias overrides**: Specific aliases can have different proxy configurations

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
- Verify alias isn't overriding project defaults
- Confirm target URL is HTTPS and reachable

**Getting CORS errors still?**
- Ensure the proxy rule is active on the alias being accessed
- Check browser DevTools for actual request URL (may not be hitting proxy)
