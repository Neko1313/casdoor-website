---
title: FastAPI
description: Using Casdoor OAuth2 authentication and Casbin authorization in a FastAPI project via casbin-fastapi-decorator.
keywords: [FastAPI, Casbin, Python, authorization, OAuth2]
authors: [Neko1313]
---

[casbin-fastapi-decorator](https://github.com/Neko1313/casbin-fastapi-decorator) is a community library that provides decorator-based Casbin authorization for FastAPI. Its **Casdoor extra** (`casbin-fastapi-decorator-casdoor`) combines Casdoor OAuth2 login with Casbin policy enforcement through Casdoor's remote enforce API — all without polluting your route signatures with dependency injection.

## Step 1: Deploy Casdoor

Deploy Casdoor in **production mode**. See [Server installation](/docs/basic/server-installation). Make sure the server is reachable and you can sign in (e.g. `admin` / `123`).

## Step 2: Configure a Casdoor application

In the Casdoor admin panel, create or open an application and configure:

- **Redirect URL** — add your FastAPI callback URL, e.g. `http://localhost:8080/callback`
- Copy the **Client ID**, **Client Secret**, and **Certificate**

You will also need either an **Enforcer**, **Permission**, or **Model** identifier to pass as the enforce target.

## Step 3: Install the library

```bash
pip install "casbin-fastapi-decorator[casdoor]"
```

Requires Python ≥ 3.10.

## Step 4: Quick start with `CasdoorIntegration`

`CasdoorIntegration` is a facade that wires together the Casdoor SDK, user provider, enforcer provider, and OAuth2 router in a single call.

```python
from casbin_fastapi_decorator_casdoor import CasdoorIntegration, CasdoorEnforceTarget
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

# Register GET /callback and POST /logout routes
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

:::danger Security: state parameter is not validated by default

`CasdoorIntegration` accepts the `state` query parameter on the OAuth2 callback but **does not verify it**. Without state validation the callback endpoint has no CSRF protection: an attacker can craft a malicious authorization URL, trick a victim into opening it, and complete the callback in the victim's browser — potentially binding the wrong Casdoor account to the victim's session.

**Do not deploy to production without addressing this.** Implement state validation before registering the router:

```python
import secrets
from fastapi import Request, HTTPException

# 1. On login redirect — generate and store state in the session
state = secrets.token_urlsafe(32)
request.session["oauth_state"] = state

# 2. On callback — verify before proceeding
incoming_state = request.query_params.get("state")
if not secrets.compare_digest(incoming_state or "", request.session.pop("oauth_state", "")):
    raise HTTPException(400, "Invalid state parameter")
```

Use a server-side session (e.g. `itsdangerous`-signed cookie or Redis) so the state cannot be forged by the client.

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

For cases requiring a custom user factory, per-route enforce targets, or custom error handling, the components can be composed directly:

```python
from casdoor import AsyncCasdoorSDK
from fastapi import FastAPI, HTTPException
from casbin_fastapi_decorator import PermissionGuard
from casbin_fastapi_decorator_casdoor import (
    CasdoorUserProvider,
    CasdoorEnforcerProvider,
    CasdoorEnforceTarget,
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
router = make_casdoor_router(sdk=sdk, redirect_after_login="/dashboard")

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
| `CasdoorIntegration` | Facade combining all of the above with configurable cookie and redirect settings |
| `make_casdoor_router` | Creates an `APIRouter` with `GET /callback` (OAuth2 code exchange) and `POST /logout` (cookie cleanup) |

## More resources

- [casbin-fastapi-decorator on GitHub](https://github.com/Neko1313/casbin-fastapi-decorator)
- [casbin-fastapi-decorator on PyPI](https://pypi.org/project/casbin-fastapi-decorator/)
- [casbin-fastapi-decorator-casdoor on PyPI](https://pypi.org/project/casbin-fastapi-decorator-casdoor/)
