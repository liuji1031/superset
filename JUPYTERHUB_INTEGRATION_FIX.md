# JupyterHub Integration Fix for Apache Superset

## Summary

This document describes the modifications made to Apache Superset to enable proper operation within JupyterHub's dynamic per-user URL prefix environment (e.g., `/user/username@example.com/`).

## The Problem

When running Superset behind JupyterHub's reverse proxy with dynamic prefixes:
- API calls were going to incorrect paths (e.g., `/api/v1/...` instead of `/user/username/api/v1/...`)
- Static assets failed to load from correct paths
- Post-login redirects went to wrong URLs (e.g., `/hub/superset/welcome/` instead of `/user/username/superset/welcome/`)

## The Solution

Two complementary modifications:

### 1. Backend Fix: Bootstrap Data Injection

**File:** `superset/views/base.py` (lines 500-507)

**What it does:** Directly reads `JUPYTERHUB_SERVICE_PREFIX` environment variable and injects it into the bootstrap data as `application_root` and `static_assets_prefix`, ensuring the frontend receives the correct prefix.

```python
# JUPYTERHUB: Prefer JUPYTERHUB_SERVICE_PREFIX env var for dynamic prefixes
# This ensures bootstrap data includes the correct prefix even if APPLICATION_ROOT
# hasn't been set yet by AppRootMiddleware
jupyterhub_prefix = os.environ.get('JUPYTERHUB_SERVICE_PREFIX', '').rstrip('/')
bootstrap_data = {
    "application_root": jupyterhub_prefix or app.config.get("APPLICATION_ROOT", ""),
    "static_assets_prefix": jupyterhub_prefix or app.config.get("STATIC_ASSETS_PREFIX", ""),
    "conf": frontend_config,
    # ... rest of bootstrap data
}
```

**Why this works:** By reading directly from the environment variable, we ensure the bootstrap data always includes the JupyterHub prefix, regardless of timing issues with middleware initialization.

### 2. Frontend Fix: SupersetClient Configuration

**File:** `superset-frontend/src/setup/setupClient.ts` (line 70)

**What it does:** Configures the `SupersetClient` to use the `application_root` from bootstrap data, ensuring all API calls are prefixed correctly.

```typescript
return {
  protocol: ['http:', 'https:'].includes(window?.location?.protocol)
    ? (window?.location?.protocol as 'http:' | 'https:')
    : undefined,
  host: window.location?.host || '',
  appRoot: bootstrapData.common.application_root || '',  // <- THE KEY LINE
  csrfToken: csrfToken || cookieCSRFToken,
  fetchRetryOptions,
};
```

**Why this works:** The `SupersetClient` class automatically prepends the `appRoot` value to all API calls via its internal `getUrl()` method.

## How It Works Together

1. **JupyterHub** spawns the Superset container with `JUPYTERHUB_SERVICE_PREFIX=/user/username/`
2. **Backend** reads this env var and injects it into bootstrap data
3. **Frontend** reads bootstrap data and configures `SupersetClient` with the prefix
4. **All API calls** automatically include the correct prefix
5. **Static assets** load from the correct prefixed paths
6. **Redirects** use the correct prefix via Flask's `url_for()` with `LOGO_TARGET_PATH`

## Deployment Configuration

### Environment Variable Setup

**File:** `deploy/Docker/app-stacks/superset/start-dashboard.sh`

The startup script converts `JUPYTERHUB_SERVICE_PREFIX` to `SUPERSET_APP_ROOT`:

```bash
if [ -n "${JUPYTERHUB_SERVICE_PREFIX:-}" ]; then
    PREFIX="${JUPYTERHUB_SERVICE_PREFIX%/}"
    export SUPERSET_APP_ROOT="${PREFIX}"
fi
```

### Flask Configuration

**File:** `deploy/Docker/app-stacks/superset/superset_config.py`

Key configurations:
- `LOGO_TARGET_PATH = "/superset/welcome/"` - Relative path (Flask prepends APPLICATION_ROOT)
- `ENABLE_PROXY_FIX = True` - Trusts X-Forwarded-* headers
- `ProxyFix` middleware - Handles reverse proxy headers

### Proxy Architecture

```
JupyterHub → Nginx (8888) → Gunicorn (8081) → Superset (Flask)
```

- **Nginx** proxies requests and forwards headers
- **Gunicorn** runs the WSGI app
- **Superset** uses AppRootMiddleware + our custom bootstrap injection

## Testing the Fix

After building and deploying, verify:

1. **Check bootstrap data in browser console:**
   ```javascript
   window.bootstrapData.common.application_root
   // Should show: "/user/username/"
   ```

2. **Check API calls in Network tab:**
   - Should show: `/user/username/api/v1/dashboard/`
   - NOT: `/api/v1/dashboard/` or `/hub/api/v1/dashboard/`

3. **Check static assets:**
   - Should load from: `/user/username/static/assets/...`
   - NOT: `/static/assets/...` or `/hub/static/assets/...`

4. **Check post-login redirect:**
   - Should redirect to: `/user/username/superset/welcome/`
   - NOT: `/hub/superset/welcome/` or `/superset/welcome/`

## Debugging

If issues persist, check the startup logs for:

```
DEBUG: Environment Variables
JUPYTERHUB_SERVICE_PREFIX: '/user/username/'
...
SUPERSET CONFIG DEBUG:
  JUPYTERHUB_SERVICE_PREFIX (env): /user/username/
  SUPERSET_APP_ROOT (env): /user/username/
  APPLICATION_ROOT (Flask config): /user/username/
```

## Why Not Use AppRootMiddleware Alone?

Superset's built-in `AppRootMiddleware` reads `SUPERSET_APP_ROOT` and sets `APPLICATION_ROOT` config. However, bootstrap data is generated early in the app initialization, potentially before middleware has fully configured the Flask app. By reading directly from the environment variable in the bootstrap data function, we ensure consistency.

## Advantages of This Approach

✅ **Minimal code changes** - Only 2 locations modified  
✅ **Uses Superset's existing mechanisms** - `SupersetClient.appRoot` is built-in  
✅ **No runtime patches needed** - Clean source code modification  
✅ **Maintainable** - Works with future Superset versions  
✅ **Upstream-able** - Could be contributed back to Apache Superset

## Files Modified

- `superset/views/base.py` - Backend bootstrap data injection
- `superset-frontend/src/setup/setupClient.ts` - Frontend client configuration

## Repository

This forked version with JupyterHub fixes: https://github.com/liuji1031/superset.git

