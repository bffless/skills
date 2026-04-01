---
name: authentication
description: Cross-domain authentication using the admin login relay pattern, built-in /_bffless/auth endpoints, and cookie-based sessions
---

# Authentication

BFFless uses a cross-domain authentication relay pattern. Users authenticate at the workspace's admin domain (`admin.<workspace>`) and are relayed back to the content domain with auth cookies. Auth endpoints on the content domain are accessed via the **built-in `/_bffless/auth/*` endpoints** — no proxy rules required.

## How Authentication Works

### Workspace Subdomains

For workspace subdomains (e.g., `myalias.sandbox.workspace.bffless.app`), SuperTokens session cookies (`sAccessToken`) work directly because they share the parent domain.

When a user visits a private deployment and isn't authenticated:

1. Backend redirects to `https://admin.<workspace>/login?redirect=<original-path>&tryRefresh=true`
2. The login page attempts a session refresh first (the `tryRefresh` param)
3. If refresh fails, the user logs in normally
4. After login, the user is redirected back to the original path
5. The `sAccessToken` cookie is valid across all subdomains of the workspace

### Custom Domains (customDomainRelay)

For custom domains (e.g., `www.bffless.com`), SuperTokens cookies don't work because they're on a completely different domain. BFFless uses a **domain relay** flow:

1. User visits a private page on `www.bffless.com/portal/`
2. Frontend detects the user is not authenticated (via `/_bffless/auth/session`)
3. Frontend redirects to the admin login with relay params:
   ```
   https://admin.console.bffless.app/login?customDomainRelay=true&targetDomain=www.bffless.com&redirect=%2Fportal%2F
   ```
4. User logs in on the admin domain (or is already logged in via SuperTokens session)
5. After login, the frontend calls `POST /api/auth/domain-token` with:
   ```json
   { "targetDomain": "www.bffless.com", "redirectPath": "/portal/" }
   ```
6. Backend validates that `targetDomain` is a registered domain for this workspace, then creates a short-lived JWT (the "domain token")
7. Backend returns a `redirectUrl` pointing to the callback on the custom domain: `https://www.bffless.com/_bffless/auth/callback?token=...&redirect=/portal/`
8. The callback endpoint validates the token, sets `bffless_access` and `bffless_refresh` HttpOnly cookies, and redirects to the original path

### Important: Use `/_bffless/auth/*`, NOT `/api/auth/*`

The `/_bffless/auth/*` endpoints are **built into BFFless nginx** and handled by a dedicated controller. They are separate from the SuperTokens `/api/auth/*` endpoints. Do NOT use `/api/auth/*` on custom domains — those are SuperTokens endpoints that use different cookies (`sAccessToken`) which are not set by the domain relay flow.

The domain relay callback sets `bffless_access` and `bffless_refresh` cookies, which are only recognized by the `/_bffless/auth/*` endpoints. Using `/api/auth/session` instead of `/_bffless/auth/session` will cause a redirect loop because the SuperTokens session check won't find the `bffless_access` cookie.

## Auth Endpoints (Built-in)

All auth endpoints are available at `/_bffless/auth/*` on any domain served by BFFless — no proxy rules needed.

| Endpoint                    | Method | Purpose                                                  |
| --------------------------- | ------ | -------------------------------------------------------- |
| `/_bffless/auth/session`    | GET    | Check current session (returns user info or 401)         |
| `/_bffless/auth/refresh`    | POST   | Refresh an expired access token using the refresh cookie |
| `/_bffless/auth/callback`   | GET    | Exchange a domain relay token for auth cookies           |
| `/_bffless/auth/logout`     | POST   | Clear auth cookies                                       |

### Session Check Priority

The `/_bffless/auth/session` endpoint checks auth in this order:

1. **`bffless_access` cookie** — custom domain JWT issued by the callback flow
2. **`sAccessToken` cookie** — SuperTokens session (fallback for workspace subdomains)

If the access token is expired, it returns `401` with `"try refresh token"` to signal the client should call `/_bffless/auth/refresh`.

## Frontend Integration

### Checking Session (with automatic token refresh)

Use a shared promise pattern to avoid duplicate session checks across components:

```typescript
async function checkSession() {
  // Reuse shared session promise so multiple components don't duplicate requests
  if (!(window as any).__bfflessSession) {
    (window as any).__bfflessSession = (async () => {
      const res = await fetch('/_bffless/auth/session', { credentials: 'include' });
      if (res.ok) return res.json();

      if (res.status === 401) {
        // Token expired — try refreshing
        const refreshRes = await fetch('/_bffless/auth/refresh', {
          method: 'POST',
          credentials: 'include',
        });
        if (refreshRes.ok) {
          // Retry session check with new token
          const retryRes = await fetch('/_bffless/auth/session', { credentials: 'include' });
          if (retryRes.ok) return retryRes.json();
        }
      }
      return null;
    })().catch(() => null);
  }

  return (window as any).__bfflessSession;
}

// Returns: { authenticated: true, user: { id, email, role } } or null
```

The flow is: session check → if 401, refresh token → retry session check. This handles the common case where the access token has expired but the refresh token is still valid.

### Redirecting to Login

When unauthenticated, redirect the user to the admin login with relay params. Use the **promoted admin domain** (e.g., `admin.console.bffless.app`), not the full workspace subdomain:

```typescript
function getLoginUrl(adminLoginUrl: string, redirectPath: string): string {
  const targetDomain = window.location.hostname;
  const params = new URLSearchParams({
    customDomainRelay: 'true',
    targetDomain,
    redirect: redirectPath,
  });
  return `${adminLoginUrl}?${params.toString()}`;
}

// Example: redirect to admin login, then relay back to /portal/
const session = await checkSession();
if (!session) {
  window.location.href = getLoginUrl(
    'https://admin.console.bffless.app/login',
    '/portal/',
  );
}
```

### Logout

```typescript
await fetch('/_bffless/auth/logout', { method: 'POST', credentials: 'include' });
```

### Updating UI Based on Auth State (Header example)

```typescript
// Check auth state and update Login/Portal links
window.__bfflessSession = window.__bfflessSession || checkBfflessSession().catch(() => null);

window.__bfflessSession.then((data) => {
  if (data?.authenticated) {
    // User is logged in — update nav links
    document.querySelectorAll('[data-auth-link]').forEach((el) => {
      el.textContent = 'Portal';
    });
  }
});
```

## Auth Flow Diagram

```
Custom Domain Flow:
┌──────────────────┐     JS redirect        ┌──────────────────────────┐
│  www.bffless.com │ ──────────────────→    │  admin.<workspace>/login │
│  (private page)  │  customDomainRelay=    │  ?customDomainRelay=true │
│                  │  true&targetDomain=    │  &targetDomain=www...    │
└──────────────────┘  www.bffless.com       └────────────┬─────────────┘
        ▲                                                │
        │                                     User logs in (SuperTokens)
        │                                                │
        │                                                ▼
        │                                   POST /api/auth/domain-token
        │                                   → returns { token, redirectUrl }
        │                                                │
        │              302 redirect                      │
        │  ←─────────────────────────────────────────────┘
        │  to: www.bffless.com/_bffless/auth/callback?token=...
        │
        ▼
┌──────────────────┐
│  /_bffless/auth  │  Validates token, sets bffless_access
│  /callback       │  + bffless_refresh cookies
│  (built-in)      │  → 302 redirect to /portal/
└──────────────────┘
```

## Troubleshooting

**User gets stuck in a redirect loop?**

- **Most common cause:** Using `/api/auth/session` instead of `/_bffless/auth/session`. The domain relay callback sets `bffless_access` cookies which are only recognized by `/_bffless/auth/*` endpoints. The `/api/auth/*` endpoints check SuperTokens cookies (`sAccessToken`) which are NOT set by the domain relay flow.
- Verify the custom domain is registered in `domain_mappings` with `isActive = true`
- Ensure cookies are being set (requires HTTPS for `Secure` flag)

**"Domain not registered" error on domain-token?**

- The `targetDomain` must match a `domain_mappings` entry or be a subdomain of `PRIMARY_DOMAIN`
- Check for www vs non-www mismatch

**Session check returns 401 but user just logged in?**

- On custom domains: verify the `/_bffless/auth/callback` was reached and cookies were set
- On workspace subdomains: verify `COOKIE_DOMAIN` is configured for cross-subdomain cookie sharing
- Check that the `bffless_access` or `sAccessToken` cookie is present in the request

**Admin login URL — use promoted domain, not workspace subdomain:**

- If the workspace has a promoted domain (e.g., `console.bffless.app`), use `admin.console.bffless.app`, NOT `admin.console.workspace.bffless.app`
- The workspace subdomain format still works but the promoted domain is cleaner
