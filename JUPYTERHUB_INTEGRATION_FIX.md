# JupyterHub Integration for Apache Superset

## Summary

This document describes the modifications made to Apache Superset to enable proper operation within JupyterHub's dynamic per-user URL prefix environment (e.g., `/user/username@example.com/`) using `jhsingle-native-proxy`.

## The Problem

When running Superset behind JupyterHub with dynamic user-specific URL prefixes:
- API calls go to incorrect paths (e.g., `/api/v1/...` instead of `/user/username/api/v1/...`)
- Static assets fail to load from correct prefixed paths
- Post-login redirects go to wrong URLs (e.g., `/superset/welcome/` instead of `/user/username/superset/welcome/`)
- User menu items (Info, Logout) have missing URL prefixes

## Architecture

### jhsingle-native-proxy Deployment

```
JupyterHub (port 8000)
  ↓ Request: /user/polus@example.com/superset/welcome/
  ↓
jhsingle-native-proxy (port 8888) ← JupyterHub spawns this
  ↓ strips /user/polus@example.com/ prefix from REQUESTS
  ↓ handles OAuth authentication
  ↓ Request: /superset/welcome/ (prefix stripped)
  ↓
Gunicorn (dynamic port) ← jhsingle manages this
  ↓ Flask serves from / (no APPLICATION_ROOT)
  ↓ receives /superset/welcome/, matches route ✓
  ↓ generates URLs and redirects
  ↓
PrefixRedirectMiddleware ← custom WSGI middleware (in superset_config.py)
  ↓ intercepts redirect responses
  ↓ adds prefix: Location: /user/polus@example.com/superset/welcome/
  ↓
Patched url_for() ← Flask and Jinja url_for overrides (in superset_config.py)
  ↓ all generated URLs include prefix
  ↓
jhsingle-native-proxy
  ↓ passes responses through unchanged
  ↓
JupyterHub → Browser ✓
```

**CRITICAL UNDERSTANDING**:
- `jhsingle-native-proxy` **strips the prefix from incoming REQUESTS**
- `jhsingle-native-proxy` **does NOT rewrite Location headers in RESPONSES**
- Flask must serve from `/` (no `APPLICATION_ROOT`) because requests arrive without prefix
- Outgoing URLs and redirects must include the prefix for browser navigation

**Solution Strategy**:
1. **Route matching**: Flask serves from `/` (jhsingle strips prefix from requests)
2. **URL generation**: Custom middleware and patched `url_for()` add prefix to all outgoing URLs
3. **Bootstrap data**: Inject prefix into frontend bootstrap data for API calls and static assets

## Source Code Modifications

### 1. Frontend: SupersetClient Configuration

**File:** `superset-frontend/src/setup/setupClient.ts` (line 70)

**Change:** Configure `SupersetClient` to use `application_root` from bootstrap data

```typescript
return {
  protocol: ['http:', 'https:'].includes(window?.location?.protocol)
    ? (window?.location?.protocol as 'http:' | 'https:')
    : undefined,
  host: window.location?.host || '',
  appRoot: bootstrapData.common.application_root || '',  // ← ADDED THIS LINE
  csrfToken: csrfToken || cookieCSRFToken,
  fetchRetryOptions,
};
```

**Why this works:** `SupersetClient` automatically prepends `appRoot` to all API calls via its internal `getUrl()` method.

---

### 2. Backend: Bootstrap Data Injection

**File:** `superset/views/base.py` (function `cached_common_bootstrap_data`)

**Change:** Read `JUPYTERHUB_SERVICE_PREFIX` and inject into bootstrap data

```python
# JUPYTERHUB: Inject application_root and static_assets_prefix from environment
jupyterhub_prefix = os.environ.get('JUPYTERHUB_SERVICE_PREFIX', '').rstrip('/')

bootstrap_data = {
    "conf": frontend_config,
    "application_root": jupyterhub_prefix or conf.get("APPLICATION_ROOT", ""),
    "static_assets_prefix": jupyterhub_prefix or conf.get("STATIC_ASSETS_PREFIX", ""),
    # ... rest of bootstrap data
}
```

**Why this works:** Ensures frontend receives correct prefix for API calls and static asset loading, even before middleware initialization.

---

### 3. Backend: User Menu URL Fixes

**File:** `superset/views/base.py` (function `menu_data`)

**Change:** Manually add JupyterHub prefix to user menu URLs that don't use `url_for()`

```python
def menu_data(user: User) -> dict[str, Any]:
    # Get JupyterHub prefix from environment for user menu URLs
    jupyterhub_prefix = os.environ.get('JUPYTERHUB_SERVICE_PREFIX', '').rstrip('/')
    
    # ... other code ...
    
    # Add JupyterHub prefix to user menu URLs (these are hardcoded and don't use url_for)
    user_info_url = None
    if not is_feature_enabled("MENU_HIDE_USER_INFO"):
        user_info_url = f"{jupyterhub_prefix}/user_info/" if jupyterhub_prefix else "/user_info/"
    
    user_logout_url = appbuilder.get_url_for_logout
    if jupyterhub_prefix and user_logout_url.startswith("/") and not user_logout_url.startswith(jupyterhub_prefix):
        user_logout_url = jupyterhub_prefix + user_logout_url
        
    user_login_url = appbuilder.get_url_for_login
    if jupyterhub_prefix and user_login_url.startswith("/") and not user_login_url.startswith(jupyterhub_prefix):
        user_login_url = jupyterhub_prefix + user_login_url

    return {
        "menu": appbuilder.menu.get_data(),
        # ... other data ...
        "navbar_right": {
            # ... other fields ...
            "user_info_url": user_info_url,
            "user_logout_url": user_logout_url,
            "user_login_url": user_login_url,
            # ... other fields ...
        },
    }
```

**Why this is needed:** Flask-AppBuilder's menu system generates these URLs without using Flask's `url_for()`, so they miss the prefix. We manually add it.

---

### 4. Backend: Login Redirect Handling

**File:** `superset/views/auth.py` (class `SupersetAuthView`)

**Change:** Enhanced login method to properly handle JupyterHub prefix in `next` parameter

```python
@expose("/", methods=["GET", "POST"])
@no_cache
def login(self, provider: Optional[str] = None) -> WerkzeugResponse:
    # Get JupyterHub prefix from environment or Flask config
    jupyterhub_prefix = os.environ.get('JUPYTERHUB_SERVICE_PREFIX', '').rstrip('/')
    if not jupyterhub_prefix:
        jupyterhub_prefix = self.appbuilder.app.config.get('APPLICATION_ROOT', '').rstrip('/')
    
    # If already authenticated, redirect appropriately
    if g.user is not None and g.user.is_authenticated:
        next_url = request.args.get("next") or request.form.get("next")
        if next_url:
            # Prepend JupyterHub prefix if needed
            if next_url.startswith("/") and not next_url.startswith(jupyterhub_prefix):
                next_url = jupyterhub_prefix + next_url
            return redirect(next_url)
        return redirect(self.appbuilder.get_url_for_index)
    
    # Handle POST (login form submission)
    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        
        if username and password:
            user = self.appbuilder.sm.auth_user_db(username, password)
            if user:
                login_user(user, remember=False)
                
                # Get next URL and add prefix if needed
                next_url = request.args.get("next") or request.form.get("next")
                if next_url and next_url.startswith("/"):
                    if not next_url.startswith(jupyterhub_prefix):
                        next_url = jupyterhub_prefix + next_url
                    return redirect(next_url)
                
                return redirect(self.appbuilder.get_url_for_index)
    
    # GET request or failed login - render login page
    return super().render_app_template()
```

**Why this is needed:** Flask-AppBuilder's default login view ignores the `next` parameter and doesn't handle the JupyterHub prefix correctly.

---

### 5. Frontend: Dataset Creation UI Fixes

**Files:** 
- `superset-frontend/src/features/datasets/AddDataset/Footer/index.tsx`
- `superset-frontend/src/features/datasets/AddDataset/LeftPanel/index.tsx`

**Change:** Use `SupersetClient.getUrl()` instead of hardcoded paths for dataset creation links

```typescript
// Before
href="/tablemodelview/list/"

// After  
href={SupersetClient.getUrl('/tablemodelview/list/')}
```

**Why this is needed:** Ensures dataset creation UI links include the JupyterHub prefix.

---

## Deployment Configuration

### Environment Variables (Set by JupyterHub)

| Variable | Example | Description |
|----------|---------|-------------|
| `JUPYTERHUB_SERVICE_PREFIX` | `/user/polus@example.com/` | User-specific URL prefix |
| `JUPYTERHUB_USER` | `polus@example.com` | Current JupyterHub user |
| `JUPYTERHUB_API_TOKEN` | `<token>` | OAuth token for JupyterHub API |
| `JUPYTERHUB_BASE_URL` | `/` | JupyterHub base URL |

### Superset Configuration

**File:** `deploy/Docker/app-stacks/superset_test/superset_config.py`

**Key configurations:**

```python
# Get JupyterHub prefix
JUPYTERHUB_SERVICE_PREFIX = os.environ.get('JUPYTERHUB_SERVICE_PREFIX', '').rstrip('/')

# Logo target should be relative (Flask adds prefix via url_for)
LOGO_TARGET_PATH = "/superset/welcome/"

# Set static assets prefix to JupyterHub prefix only
if JUPYTERHUB_SERVICE_PREFIX:
    STATIC_ASSETS_PREFIX = JUPYTERHUB_SERVICE_PREFIX

# Custom middleware to add prefix to Location headers in redirects
class PrefixRedirectMiddleware:
    def __init__(self, app, prefix):
        self.app = app
        self.prefix = prefix.rstrip('/')
    
    def __call__(self, environ, start_response):
        def custom_start_response(status, headers, exc_info=None):
            if status.startswith(('301', '302', '303', '307', '308')):
                modified_headers = []
                for name, value in headers:
                    if name.lower() == 'location':
                        if value.startswith('/') and not value.startswith(self.prefix):
                            value = self.prefix + value
                    modified_headers.append((name, value))
                headers = modified_headers
            return start_response(status, headers, exc_info)
        return self.app(environ, custom_start_response)

# Flask app mutator
def FLASK_APP_MUTATOR(app):
    # ProxyFix for X-Forwarded headers (x_prefix=0 because jhsingle doesn't send it)
    app.wsgi_app = ProxyFix(
        app.wsgi_app, x_for=1, x_proto=1, x_host=1, x_port=1, x_prefix=0
    )
    
    jupyterhub_prefix = os.environ.get('JUPYTERHUB_SERVICE_PREFIX', '').rstrip('/')
    if jupyterhub_prefix:
        # Add redirect middleware
        app.wsgi_app = PrefixRedirectMiddleware(app.wsgi_app, jupyterhub_prefix)
        
        # Patch Flask's url_for
        from flask import url_for as flask_url_for
        def prefixed_flask_url_for(endpoint, **values):
            url = flask_url_for(endpoint, **values)
            if url.startswith('/') and not url.startswith(jupyterhub_prefix):
                url = jupyterhub_prefix + url
            return url
        import flask
        flask.url_for = prefixed_flask_url_for
        
        # Patch Jinja's url_for
        original_jinja_url_for = app.jinja_env.globals['url_for']
        def prefixed_jinja_url_for(endpoint, **values):
            url = original_jinja_url_for(endpoint, **values)
            if url.startswith('/') and not url.startswith(jupyterhub_prefix):
                url = jupyterhub_prefix + url
            return url
        app.jinja_env.globals['url_for'] = prefixed_jinja_url_for
```

**Why this approach:**
- **No `APPLICATION_ROOT`**: Flask serves from `/` because jhsingle strips the prefix from requests
- **Custom middleware**: Adds prefix to `Location` headers in redirect responses
- **Patched `url_for()`**: Ensures all Flask-generated URLs include the prefix
- **`STATIC_ASSETS_PREFIX`**: Set to prefix so static assets load from correct path

### Launch Command

**File:** `deploy/Docker/app-stacks/superset_test/start-dashboard.sh`

```bash
# Use standard jhsingle command pattern
$JHSINGLE_COMMAND \
    --destport 0 \
    --ready-check-path /health \
    --ready-timeout 60 \
    -- \
    gunicorn \
        --bind "127.0.0.1:{port}" \
        --workers "$GUNICORN_WORKERS" \
        --threads "$GUNICORN_THREADS" \
        --timeout "$GUNICORN_TIMEOUT" \
        "superset.app:create_app()"
```

**Why Gunicorn:**
- Flask's dev server is single-threaded and not production-ready
- Gunicorn provides multiple workers/threads for better performance
- Required for proper WSGI application serving

## How It Works Together

1. **JupyterHub** spawns container with `JUPYTERHUB_SERVICE_PREFIX=/user/username/`
2. **jhsingle-native-proxy** starts, strips prefix from incoming requests, handles OAuth
3. **Backend** (via `superset_config.py`):
   - Reads `JUPYTERHUB_SERVICE_PREFIX` environment variable
   - Configures `STATIC_ASSETS_PREFIX` for frontend asset loading
   - Installs `PrefixRedirectMiddleware` to add prefix to redirect responses
   - Patches `url_for()` to add prefix to all generated URLs
4. **Backend** (via `superset/views/base.py`):
   - Injects prefix into bootstrap data as `application_root` and `static_assets_prefix`
   - Manually adds prefix to user menu URLs (Info, Logout)
5. **Frontend** (via `setupClient.ts`):
   - Reads `application_root` from bootstrap data
   - Configures `SupersetClient` to prepend prefix to all API calls
6. **Frontend** (via `public-path.ts`):
   - Uses `static_assets_prefix` from bootstrap data for webpack public path
   - All static assets (JS, CSS, images) load from correct prefixed path

## Testing the Integration

### 1. Verify Bootstrap Data

Open browser console after logging in:
```javascript
window.bootstrapData.common.application_root
// Should show: "/user/username/"

window.bootstrapData.common.static_assets_prefix
// Should show: "/user/username/"
```

### 2. Verify API Calls

Check Network tab in browser DevTools:
- ✅ Should show: `/user/username/api/v1/dashboard/`
- ❌ NOT: `/api/v1/dashboard/` or `/hub/api/v1/dashboard/`

### 3. Verify Static Assets

Check loaded resources in Network tab:
- ✅ Should load from: `/user/username/static/assets/...`
- ❌ NOT: `/static/assets/...` or `/hub/static/assets/...`

### 4. Verify Redirects

After login:
- ✅ Should redirect to: `/user/username/superset/welcome/`
- ❌ NOT: `/superset/welcome/` or `/hub/superset/welcome/`

### 5. Verify User Menu

Click on user menu in top right:
- Info link: ✅ `/user/username/user_info/`
- Logout link: ✅ `/user/username/logout/`

## Debugging

### Check Environment Variables

```bash
docker exec <container-id> env | grep JUPYTERHUB
```

Expected output:
```
JUPYTERHUB_SERVICE_PREFIX=/user/username/
JUPYTERHUB_USER=username
JUPYTERHUB_API_TOKEN=<token>
JUPYTERHUB_BASE_URL=/
```

### Check Startup Logs

```bash
docker logs <container-id> | grep -A 5 "SUPERSET CONFIG"
```

Expected output:
```
SUPERSET CONFIG DEBUG:
  JUPYTERHUB_SERVICE_PREFIX (env): /user/username/
  STATIC_ASSETS_PREFIX (Flask config): /user/username/
  LOGO_TARGET_PATH: /superset/welcome/
```

### Check Gunicorn Configuration

```bash
docker logs <container-id> | grep "Gunicorn config"
```

Expected output:
```
Gunicorn config: workers=4, threads=4, timeout=60
```

## Files Modified Summary

### Backend (Python)
1. **`superset/views/base.py`**
   - Modified `cached_common_bootstrap_data()` to inject prefix into bootstrap data
   - Modified `menu_data()` to add prefix to user menu URLs

2. **`superset/views/auth.py`**
   - Created `SupersetAuthView` class to override login behavior
   - Added proper handling of `next` parameter with JupyterHub prefix

### Frontend (TypeScript/TSX)
3. **`superset-frontend/src/setup/setupClient.ts`**
   - Added `appRoot: bootstrapData.common.application_root` to SupersetClient config

4. **`superset-frontend/src/features/datasets/AddDataset/Footer/index.tsx`**
   - Changed hardcoded paths to use `SupersetClient.getUrl()`

5. **`superset-frontend/src/features/datasets/AddDataset/LeftPanel/index.tsx`**
   - Changed hardcoded paths to use `SupersetClient.getUrl()`

### Configuration (Deployment)
6. **`deploy/Docker/app-stacks/superset_test/superset_config.py`** (not in Superset repo)
   - Custom `PrefixRedirectMiddleware` class
   - `FLASK_APP_MUTATOR` with patched `url_for()` functions
   - `STATIC_ASSETS_PREFIX` configuration

7. **`deploy/Docker/app-stacks/superset_test/start-dashboard.sh`** (not in Superset repo)
   - Launches via `jhsingle-native-proxy` with Gunicorn

## Advantages of This Approach

✅ **Minimal source code changes** - Only 5 files in Superset repository  
✅ **Standard jhsingle pattern** - Matches other dashboard deployments  
✅ **No nginx configuration** - Simpler deployment stack  
✅ **Configurable performance** - Gunicorn workers/threads via environment variables  
✅ **Built-in OAuth** - jhsingle handles JupyterHub authentication  
✅ **Maintainable** - Clear separation between Superset source and deployment config  
✅ **Upstream-able** - Source changes could potentially be contributed to Apache Superset

## Repository

Forked Superset with JupyterHub integration: https://github.com/liuji1031/superset.git

**Key commits:**
- `a9bff3803`: Added bootstrap data injection (`setupClient.ts`)
- `378a0f1d8`: Modified bootstrap data generation (`views/base.py`)
- `50aae5b2c`: Fixed hardcoded URLs in dataset UI
- `a6a294088`: Login respects `next` parameter
- `ce8470815`: Login redirect fixes
- `41b330220`: User menu URL fixes

## Performance Tuning

### Environment Variables

Configure via JupyterHub spawner:

```python
c.KubeSpawner.environment = {
    'SUPERSET_GUNICORN_WORKERS': '8',      # More workers for high load
    'SUPERSET_GUNICORN_THREADS': '4',      # Threads per worker
    'SUPERSET_GUNICORN_TIMEOUT': '120',    # Timeout for slow queries
}
```

### Sizing Guidelines

| Users | Workers | Threads | CPU | Memory |
|-------|---------|---------|-----|--------|
| < 10  | 2-4     | 4       | 1-2 | 2-4 GB |
| 10-50 | 4-8     | 4       | 2-4 | 4-8 GB |
| 50+   | 8-16    | 4       | 4-8 | 8-16 GB |
