# Tools, middleware, hooks, and the context dict

The Python side of a Connic project. All four file types (`tools/*.py`, `middleware/*.py`, `hooks/*.py`, `guardrails/*.py`) are auto-discovered — no decorators, no registration, no imports of the SDK except for special exceptions or predefined tools.

## Tools (`tools/*.py`)

Plain Python functions. The runtime introspects type hints and the docstring to build the schema shown to the LLM. The docstring is what the LLM reads to decide *when* to call the tool — write it for the model, not the developer.

```python
def lookup_invoice(invoice_id: str) -> dict:
    """Look up the status and amount of an invoice.

    Args:
        invoice_id: The invoice identifier (e.g. INV-123)

    Returns:
        A dict with keys: invoice_id, status, amount, currency.
    """
    return {
        "invoice_id": invoice_id,
        "status": "paid",
        "amount": 199.0,
        "currency": "USD",
    }
```

Async is supported and preferred for I/O:

```python
import httpx

async def web_lookup(url: str, timeout: int = 10) -> dict:
    """Fetch a URL and return its body as text.

    Args:
        url: Absolute URL to fetch.
        timeout: Seconds before giving up.

    Returns:
        Dict with status_code and body.
    """
    async with httpx.AsyncClient(timeout=timeout) as client:
        r = await client.get(url)
        return {"status_code": r.status_code, "body": r.text}
```

### The `context` parameter (auto-injected)

If a tool declares a parameter **named** `context`, the runtime fills it in and the LLM never sees it. Injection is by parameter *name*, not by type hint — `context` (no annotation) works the same as `context: dict`. Don't name an LLM-facing parameter `context` or it'll be hidden.

```python
async def get_user_orders(limit: int = 10, context: dict = {}) -> list:
    """Return recent orders for the current user.

    Args:
        limit: Max orders to return.

    Returns:
        List of order dicts.
    """
    user_id = context["user_id"]     # set earlier by middleware
    context["tool_calls"] = context.get("tool_calls", 0) + 1
    return await fetch_orders(user_id, limit)
```

`context` carries system fields (`run_id`, `agent_name`, `connector_id`, `timestamp`), the original connector `payload`, and — added after the run completes, so only readable in `after()` — `token_usage` and `duration_ms`, plus whatever middleware put there.

### Special control flow

```python
from connic import StopProcessing

def gated(resource: str, context: dict) -> str:
    """Edit a resource. Requires write access."""
    if not context.get("allow_write"):
        raise StopProcessing("Write access denied.")  # ends the run, status=completed
    return f"Edited {resource}"
```

- `StopProcessing(msg, publish_outbound=True)` — raise from a tool, middleware, or hook. Ends the *entire run* gracefully (status `completed`, the message becomes the run's output). Used when a precondition isn't met but it's not an error. Pass `publish_outbound=False` to keep outbound connectors from publishing this completed run.
- `AbortTool(result)` — **only** valid inside a `hooks/<agent>.py::before()`. Aborts just *this* tool call, returns `result` to the LLM in place of the real result, and the agent continues. Not for use from inside tool functions.
- Any other unhandled exception → tool call fails, LLM sees the error and may retry or give up.

### Logging

`print(...)` is captured at INFO level. The `logging` stdlib is captured at whatever level you used. Both appear in **Dashboard → Logs** tagged with `tools.<tool_name>`. Don't log secrets — there's no automatic redaction here.

### Environment variables

Read with `os.environ.get(...)`. Configure in **Dashboard → Project Settings → Variables** (per environment). Sensitive variables are masked in logs.

```python
import os
api_key = os.environ.get("STRIPE_API_KEY")
base_url = os.environ.get("BASE_URL", "https://api.example.com")
```

## Middleware (`middleware/<agent-name>.py`)

Auto-attached to the agent with the matching `name`. Defines an optional `before` and/or `after`. Both can be sync or async.

**Both functions must return their primary argument** — `before` returns `content`, `after` returns `response`. The runtime takes the returned value and feeds it forward (into the LLM for `before`, out to the caller for `after`). If you forget to return, the runtime gets `None` and the run fails — implicit `return None` is the single most common middleware bug, especially when you delegate the body to a helper. Modifying `content`/`context` in place is fine; you still have to return it at the end.

```python
from typing import Dict, Any
from datetime import datetime

async def before(content: Dict[str, Any], context: Dict[str, Any]) -> Dict[str, Any]:
    """Runs once before the agent reasons. Mutate content/context, then return content."""
    # Inject the current date as a leading user message part
    today = datetime.now().strftime("%A, %B %d, %Y")
    content["parts"].insert(0, {"text": f"[Today: {today}]"})

    # Put values into context for tools and {placeholders} in system_prompt
    context["user_id"] = content.get("user_id", "anonymous")
    context["tier"] = lookup_tier(context["user_id"])
    return content   # <-- REQUIRED, even if you didn't visibly change `content`


async def after(response: str, context: Dict[str, Any]) -> str:
    """Runs once after the agent finishes. Mutate response, then return it."""
    return response.strip()   # <-- REQUIRED
```

### Delegating to a helper module

When the actual work lives in a helper (`middleware/_auth.py`, `middleware/_common.py`, etc.), the `before`/`after` functions are still the contract surface — they still have to return. A common mistake is to write a thin delegate that drops the return value:

```python
# WRONG — implicit `return None`, the run will fail
from middleware._auth import authenticate

async def before(content, context):
    await authenticate(content, context)
```

```python
# RIGHT — pass `content` through after the helper mutates it/context
from middleware._auth import authenticate

async def before(content, context):
    await authenticate(content, context)   # mutates context in place
    return content
```

```python
# ALSO RIGHT — helper that explicitly returns the (possibly transformed) content
from middleware._auth import authenticate_and_normalize

async def before(content, context):
    return await authenticate_and_normalize(content, context)
```

Either shape is fine — what matters is that the value returned from `before` is whatever you want the LLM to see, and the value returned from `after` is whatever you want the caller to see. Helpers in `middleware/_*.py` files are skipped by the auto-discovery loader (the leading underscore hides them), so they're a clean place to put shared code.

### The `content` shape

`content` is the **LLM-facing message** the agent will reason over. It always has `role: "user"` and a list of `parts`; each part is either text or a binary attachment.

```python
content = {
    "role": "user",   # always "user" on input
    "parts": [
        {"text": "the user's message"},
        {"data": b"raw bytes", "mime_type": "application/pdf"},   # attached file
    ],
}
```

Attach files dynamically by appending another part:

```python
with open("docs/pricing.pdf", "rb") as f:
    content["parts"].append({"data": f.read(), "mime_type": "application/pdf"})
```

### The raw connector payload (`context["payload"]`)

`content` is **not** the raw thing the connector received — Connic has already shaped it into the LLM-facing message. The original connector payload (webhook JSON body, form fields, GET query string, Kafka message, etc.) is preserved in `context["payload"]` so middleware can read fields the LLM should never see — auth tokens, identity claims, routing metadata, idempotency keys.

```python
async def before(content, context):
    payload = context.get("payload", {})    # original, untransformed
    # For a JSON webhook body {"auth_token": "abc", "question": "Hello"}:
    #   payload["auth_token"] == "abc"
    #   payload["question"]   == "Hello"
    # For a GET webhook <url>?user_id=123&query=hello:
    #   payload["user_id"] == "123"
    #   payload["query"]   == "hello"
    return content
```

Use `context["payload"]` when you want **request metadata** that the agent shouldn't reason about — most commonly an auth token you're going to verify in middleware:

```python
from connic import StopProcessing

async def before(content, context):
    payload = context.get("payload", {})
    token = payload.get("auth_token")
    if not token:
        raise StopProcessing("Authentication required")
    context["user_id"] = verify_token(token)   # token never reaches the LLM
    return content
```

Use `content` when you want to **change what the agent sees** — attach a document, prepend customer context, redact PII before reasoning, etc. The two are independent: mutating `content` doesn't change `context["payload"]`, and vice versa.

### When middleware re-runs

By default `before` runs once. If the agent retries (due to `retry_options`), `before` only re-runs if `retry_options.rerun_middleware: true`. `after` always runs once, after all retries.

To reject a request from middleware:

```python
from connic import StopProcessing
async def before(content, context):
    if not is_valid(content):
        raise StopProcessing("Invalid request format.")
    return content
```

### End-user authentication and per-run permissions

This is a distinct concept from Connic's **team roles** (Owner / Admin / Member). Team roles control **who can sign in to the Connic dashboard and manage the project**. They have nothing to do with authenticating the end users whose requests trigger agents.

Authenticating end users — and propagating their identity, permissions, customer scope, etc. into the run — is done in `middleware/<agent>.py::before`. The connector secret authenticates the *caller* (your gateway, your app); the middleware authenticates the *end user* whose data is in the payload.

When the inbound payload already carries a per-user credential like a JWT, you can disable the connector-level secret too (the `Require Authentication` toggle on the webhook/websocket/mcp connector — default on, can be turned off). That collapses both layers into one middleware-side verification step instead of stacking a static shared secret on top of a JWT that already proves more. Leave the connector secret on when the caller is just another service of yours and there's no per-user credential to verify.

A typical pattern: your backend (or a thin proxy in front of the connector) attaches a JWT / OIDC token to the payload. Middleware verifies it, looks up permissions, and hydrates them into `context`. Conditional tools then gate sensitive actions on `context.permissions` from the agent YAML — no glue code in tools.

```python
# middleware/support-assistant.py
import os
import jwt  # PyJWT
from connic import StopProcessing

JWKS_URL = os.environ["AUTH0_JWKS_URL"]
AUDIENCE = os.environ["AUTH0_AUDIENCE"]
_jwks_client = jwt.PyJWKClient(JWKS_URL)

async def before(content, context):
    # Read the auth token from the raw connector payload — not from `content`,
    # which has already been shaped into the LLM-facing message.
    payload = context.get("payload", {})
    token = payload.get("auth_token")
    if not token:
        raise StopProcessing("Missing auth token.")
    try:
        signing_key = _jwks_client.get_signing_key_from_jwt(token).key
        claims = jwt.decode(token, signing_key, algorithms=["RS256"], audience=AUDIENCE)
    except jwt.PyJWTError as e:
        raise StopProcessing(f"Invalid auth: {e}")

    # Hydrate identity + permissions into context for tools, prompts, and conditional gates
    context["user_id"]     = claims["sub"]
    context["customer_id"] = claims.get("customer_id")
    context["permissions"] = claims.get("permissions", [])  # list[str]
    return content
```

```yaml
# agents/support-assistant.yaml — gate tools by the permissions hydrated above
tools:
  - orders.find_orders                                    # always available
  - admin.delete_order: "'orders:delete' in context.permissions"
  - billing.refund:     "'billing:refund' in context.permissions"
```

The conditional-tool expressions are plain Python evaluated against `context.*` and `input.*`, validated at deploy time — no runtime cost. See the [tools-list patterns section](agent-yaml.md#tools-list--patterns) for the full conditional-tool syntax.

Notes:

- **Don't try to map end users to Connic team roles.** Owner/Admin/Member exists for the humans who develop, deploy, and audit the project; there is no API to create a Connic "Member" per end user, and you wouldn't want one.
- **Cache external auth lookups in module scope, not in `context`.** A Redis client or a JWKS client should be a module-level singleton — `context` is per-run.
- **Identity gets recorded.** Once `context["user_id"]` is set in `before`, it shows up in run logs and traces — that's how every run becomes attributable for audit purposes (relevant for EU AI Act and similar regimes).
- **Forward identity to downstream MCP servers.** When a downstream MCP server gates on the end-user (compliance, per-tenant data, audit-trail), use `${context.<key>}` interpolation inside `mcp_servers[].headers` so each MCP call carries the right user. Connic substitutes the value per run from the same `context` you populated in `before`; missing values fail the run rather than silently sending an unauthenticated request. See the [MCP header interpolation section](guardrails-schemas-mcp.md#header-interpolation-deploy-time-vs-per-run).
- **Hooks complement, don't replace, this.** Use `hooks/<agent>.py::before` for cross-cutting checks on tool arguments (e.g. "this customer_id matches `context.customer_id`") that are easier to write once than to encode as a conditional-tool expression.

## Tool hooks (`hooks/<agent-name>.py`)

Wraps *every* tool call for the agent. Use it to redact arguments, log, or block specific tools.

```python
from connic import AbortTool

async def before(tool_name: str, params: dict, context: dict) -> dict:
    """Runs before each tool call. Return modified params (or original)."""
    if tool_name == "delete_order" and not context.get("is_admin"):
        raise AbortTool({"error": "Permission denied"})   # tool skipped, LLM sees this dict
    if "order_id" in params:
        params["order_id"] = params["order_id"].upper()
    return params


async def after(tool_name: str, params: dict, result, context: dict):
    """Runs after each tool call. Return modified result."""
    if isinstance(result, dict) and "password" in result:
        result["password"] = "***"
    return result
```

Scope: hooks fire for local tools, predefined Connic tools, and API-spec tools. They **do not** fire for tools exposed via remote MCP servers.

## Custom guardrails (`guardrails/<name>.py`)

Reference from an agent YAML — the **filename must match the `name:` value** (so `name: domain_check` → `guardrails/domain_check.py`):

```yaml
guardrails:
  input:
    - type: custom
      name: domain_check
      mode: block
```

Implementation — the signature is **exactly** `check(content, context)`. No `config` argument is passed; if you need parameterization, read from `os.environ` or hard-code per-file:

```python
from connic import GuardrailResult

ALLOWED_DOMAINS = ("acme.com", "example.com")

async def check(content: str, context: dict) -> GuardrailResult:
    if not any(d in content for d in ALLOWED_DOMAINS):
        return GuardrailResult(passed=False, message="Off-topic input.")
    return GuardrailResult(passed=True)
```

`GuardrailResult` has three fields: `passed: bool`, `message: Optional[str]`, `details: Optional[dict]`. There is no `redacted_content` field — for redact-style behavior use the built-in `pii` / `pii_leakage` types, which handle redaction themselves.

## The `context` dict — full picture

Lives for one run, shared by middleware → tools → hooks → after.

System fields (auto-populated by the runtime, available from `before`, tools, hooks, and `after`):

- `run_id` (str)
- `agent_name` (str)
- `connector_id` (str, if triggered via a connector)
- `timestamp` (ISO 8601)
- `payload` (dict) — the **original connector payload** (webhook JSON body, GET query params, form fields, Kafka message, etc.). This is the raw input before Connic shaped it into the LLM-facing `content`. Read it from anywhere — tools and `after` see the same `payload` middleware does.
- `token_usage` (dict, available in `after` and after each LLM call)
- `duration_ms` (int, available in `after`)

Anything else (`user_id`, `is_admin`, `permissions`, etc.) is whatever middleware/tools put there. Document a project's expected context keys in a README so collaborators know what's available.

## Common mistakes

- **Forgetting type hints on a tool parameter** — the parameter is still exposed but defaults to `{"type": "string"}` in the LLM-facing schema, which silently coerces numeric/boolean inputs. Always type-hint params so the LLM can pass the right types.
- **Writing tools without docstrings** — the LLM gets `"No description provided."` and won't know when to call the tool. Docstring is the primary triggering signal.
- **`_`-prefixed files and functions are skipped by the loader** — `tools/_helpers.py`, `tools/utils.py::_internal_fn`, and `__init__.py` are all invisible to the runtime. Use the leading underscore to hide module-internal helpers; drop it to expose them as tools.
- **Importing `connic` for tools that don't need it** — adds startup cost for nothing.
- **Not returning from `before` / `after`.** This is the single most common middleware bug. Both functions must return their primary argument — `return content` from `before`, `return response` from `after`. Implicit `return None` makes the run fail. Particularly easy to miss when the body delegates to a helper: `await authenticate(content, context)` on its own is *wrong* — add `return content` on the next line.
- **Sharing module-level state across runs** — runs share the Python process, so a module-level dict will leak between invocations. Use the `context` dict instead.
- **Naming an LLM-facing parameter `context`** — the runtime will hide it from the LLM and inject runtime state instead. Pick another name.
