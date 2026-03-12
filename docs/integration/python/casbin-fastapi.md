---
title: FastAPI
description: Using Casdoor OAuth2 authentication and Casbin authorization in a FastAPI project via casbin-fastapi-decorator.
keywords: [FastAPI, Casbin, Python, authorization, OAuth2]
authors: [Neko1313]
---

[casbin-fastapi-decorator](https://github.com/Neko1313/casbin-fastapi-decorator) is a community library that provides decorator-based Casbin authorization for FastAPI. Its **Casdoor extra** (`casbin-fastapi-decorator-casdoor`) combines Casdoor OAuth2 login with Casbin policy enforcement through Casdoor's remote enforce API, while keeping route signatures clean and dependency-injection free.

## Step 1: Deploy Casdoor

Deploy Casdoor in **production mode**. See [Server installation](/docs/basic/server-installation). Make sure the server is reachable and you can sign in (e.g. `admin` / `123`).

## Step 2: Configure a Casdoor application

In the Casdoor admin panel, create or open an application and configure:

- **Redirect URL** — add your FastAPI callback URL, e.g. `http://localhost:8080/callback`
- Copy the **Client ID**, **Client Secret**, and **Certificate**

You will also need either an **Enforcer**, **Permission**, or **Model** identifier to pass as the enforce target.

The application should start sign-in at `GET /login`; Casdoor itself should redirect back only to `GET /callback`.

## Step 3: Install the library

```bash
pip install "casbin-fastapi-decorator[casdoor]>=0.2.3"
```

Requires Python ≥ 3.10.

:::info Version guidance
Use `casbin-fastapi-decorator` `0.2.3` or newer in production. Versions `0.2.2` and earlier accepted the OAuth2 `state` parameter on the callback but did not validate it, which left the login flow without CSRF protection. Version `0.2.3` adds a dedicated `GET /login` entry point, default `CookieStateManager`, and callback `state` verification.
:::

## Step 4: Quick start with `CasdoorIntegration`

`CasdoorIntegration` is a facade that wires together the Casdoor SDK, user provider, enforcer provider, and OAuth2 router in a single call.

```python
from casbin_fastapi_decorator_casdoor import (
    CasdoorEnforceTarget,
    CasdoorIntegration,
)
from fastapi import FastAPI

app = FastAPI()

casdoor = CasdoorIntegration(
    endpoint="http://localhost:8000",
    client_id="<client-id>",
    client_secret="<client-secret>",
    certificate="-----BEGIN CERTIFICATE-----\n...",
    org_name="built-in",
    application_name="app-built-in",
    target=CasdoorEnforceTarget(
        enforce_id=lambda parsed: f"{parsed['owner']}/my-enforcer"
    ),
)

# Register GET /login, GET /callback and POST /logout routes
app.include_router(casdoor.router)

# Create a pre-configured PermissionGuard
guard = casdoor.create_guard()

@app.get("/protected")
@guard.require_permission("resource", "read")
async def protected():
    return {"ok": True}

@app.get("/me")
@guard.auth_required()
async def me():
    return {"ok": True}
```

Start the browser flow at `GET /login`. The router issues a one-time OAuth2 `state`, redirects to Casdoor, validates that `state` on the callback, exchanges the authorization code for tokens, and finally stores `access_token` and `refresh_token` cookies.

:::tip Custom state storage
`0.2.3` uses `CookieStateManager` by default, which is sufficient for most deployments. If you need server-side or shared state storage, pass a custom `state_manager` implementing `CasdoorStateManager`.

```python
from casbin_fastapi_decorator_casdoor import (
    CasdoorStateManager,
    CookieStateManager,
    make_casdoor_router,
)

state_manager: CasdoorStateManager = CookieStateManager(
    cookie_name="casdoor_oauth_state",
    cookie_secure=True,
)

# assuming existing_sdk = AsyncCasdoorSDK(...)
router = make_casdoor_router(
    sdk=existing_sdk,
    state_manager=state_manager,
)
```

If you are upgrading from `0.2.2` or earlier, update your sign-in links to point at `/login` instead of constructing the Casdoor authorization URL manually.
:::

## Step 5: Choose an enforce target

`CasdoorEnforceTarget` controls which Casdoor API identifier is used for policy checks. Fields accept static strings or callables that receive the parsed JWT payload:

| Field | Casdoor API parameter |
|---|---|
| `enforce_id` | Enforcer ID |
| `permission_id` | Permission ID |
| `model_id` | Model ID |
| `resource_id` | Resource ID |
| `owner` | Organization owner |

```python
# Dynamic — organisation taken from the user's JWT
CasdoorEnforceTarget(enforce_id=lambda parsed: f"{parsed['owner']}/my-enforcer")

# Static
CasdoorEnforceTarget(permission_id="built-in/can-read-articles")
```

## Advanced usage (manual composition)

For cases requiring a custom user factory, per-route enforce targets, custom `state` storage, or custom error handling, the components can be composed directly:

```python
from casdoor import AsyncCasdoorSDK
from fastapi import FastAPI, HTTPException
from casbin_fastapi_decorator import PermissionGuard
from casbin_fastapi_decorator_casdoor import (
    CasdoorEnforceTarget,
    CasdoorEnforcerProvider,
    CasdoorUserProvider,
    CookieStateManager,
    make_casdoor_router,
)

sdk = AsyncCasdoorSDK(
    endpoint="http://localhost:8000",
    client_id="<client-id>",
    client_secret="<client-secret>",
    certificate="<certificate>",
    org_name="built-in",
    application_name="app-built-in",
)

target = CasdoorEnforceTarget(permission_id="built-in/can-read")
user_provider = CasdoorUserProvider(sdk=sdk)
enforcer_provider = CasdoorEnforcerProvider(sdk=sdk, target=target)
router = make_casdoor_router(
    sdk=sdk,
    state_manager=CookieStateManager(cookie_secure=True),
    redirect_after_login="/dashboard",
)

guard = PermissionGuard(
    user_provider=user_provider,
    enforcer_provider=enforcer_provider,
    error_factory=lambda user, *rv: HTTPException(403, "Forbidden"),
)

app = FastAPI()
app.include_router(router)

@app.get("/articles")
@guard.require_permission("articles", "read")
async def list_articles():
    return []
```

## Components overview

| Component | Description |
|---|---|
| `CasdoorUserProvider` | Validates `access_token` and `refresh_token` cookies via the Casdoor SDK (JWT verification) |
| `CasdoorEnforcerProvider` | Delegates policy checks to Casdoor's remote enforce API (`/api/enforce`) |
| `CasdoorEnforceTarget` | Selects which Casdoor API identifier to use; values can be static or callables receiving the parsed JWT |
| `CookieStateManager` | Default one-time OAuth2 `state` storage and verification using an HttpOnly cookie |
| `CasdoorIntegration` | Facade combining all of the above with configurable cookie, redirect, and `state` settings |
| `make_casdoor_router` | Creates an `APIRouter` with `GET /login`, `GET /callback`, and `POST /logout` |

## More resources

- [casbin-fastapi-decorator on GitHub](https://github.com/Neko1313/casbin-fastapi-decorator)
- [casbin-fastapi-decorator on PyPI](https://pypi.org/project/casbin-fastapi-decorator/)
- [casbin-fastapi-decorator-casdoor on PyPI](https://pypi.org/project/casbin-fastapi-decorator-casdoor/)
- [casbin-fastapi-decorator changelog](https://github.com/Neko1313/casbin-fastapi-decorator/blob/main/CHANGELOG.md)
