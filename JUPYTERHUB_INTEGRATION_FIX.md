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
Flask dev server (port 8088) ← jhsingle manages this
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