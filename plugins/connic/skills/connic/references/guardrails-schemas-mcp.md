# Guardrails, output schemas, MCP, and API spec tools

These four features all live in the agent YAML and shape what the agent *can* take in, send out, or call.

## Output schemas

Force an LLM agent to emit structured JSON conforming to a JSON Schema. Files live in `schemas/<name>.json`. Reference by bare name (no extension) in YAML.

```yaml
# agents/sentiment.yaml
version: "1.0"
name: sentiment
type: llm
model: gemini/gemini-2.5-pro
description: "Classify input sentiment"
system_prompt: "Classify the user input as positive, neutral, or negative."
output_schema: sentiment-result   # schemas/sentiment-result.json
```

`schemas/sentiment-result.json`:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "sentiment": { "type": "string", "enum": ["positive", "neutral", "negative"] },
    "score":     { "type": "number" },
    "reason":    { "type": "string" }
  },
  "required": ["sentiment", "score"]
}
```

Schemas follow the JSON Schema standard. Connic honors the keywords the model needs to produce conforming output and validates the parsed response against them — invalid output causes the run to fail with a `ValidationError`. Supported:

| Category | Keywords |
| --- | --- |
| Structure | `type`, `properties`, `required`, `items`, `description`, `default`, `additionalProperties` (set to `false` to forbid extras), `nullable` / `"type": ["...", "null"]` |
| Values | `enum`, `const` |
| Numbers | `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, `multipleOf` |
| Strings | `minLength`, `maxLength`, `pattern` |
| Arrays | `minItems`, `maxItems` |

Supported types: `string`, `number`, `integer`, `boolean`, `array`, `object`, `null`.

Output schemas apply only to LLM agents.

## Guardrails

Run at request entry (input) and response exit (output). Configured under `guardrails:` in the agent YAML.

**Ordering matters.** The normal flow is input → input guardrails → `before` middleware → agent execution → `after` middleware → output guardrails → response. Input guardrails therefore cannot see values that middleware sets on `context` — `context["user_id"]`, `context["permissions"]`, anything you hydrate from a JWT — because none of it is populated yet. Input guardrails see the raw `content` string (plus auto-populated system fields such as `context["payload"]` and `run_id`). Output guardrails run after `after` middleware, so they see its final response and the fully enriched `context`.

Concrete consequences:

- **Don't do user-permission gating in a custom input guardrail** — there's no middleware-set `user_id` to check against. Either parse the credential out of `content` / `context["payload"]` inside the guardrail itself, or do the permission check in middleware (use `StopProcessing` to reject) or with conditional tools in the agent YAML. The latter two are the recommended path.
- **Reading from `context["payload"]` in an input guardrail is fine** — `payload` is a system field, populated before guardrails fire.
- **Output guardrails can use middleware-set context** for anything that depends on identity, tenancy, or per-run state.

```yaml
guardrails:
  input:
    - type: prompt_injection
      mode: block                    # block | warn
    - type: pii
      mode: redact                   # block | warn | redact
      config:
        entities: [email, phone, ssn, credit_card, iban, ip_address, api_key]
    - type: topic_restriction
      mode: block                    # block | warn (no redact)
      config:
        allowed_topics: [support, billing]
        off_topic_message: "I can only help with support or billing."
    - type: regex
      mode: block                    # block | warn (no redact)
      config:
        patterns:
          - pattern: "secret.*"
            message: "Request blocked."
    - type: moderation
      mode: block
      config:
        categories: [hate, harassment, self_harm, violence, sexual, illegal_activity, dangerous_instructions]
    - type: custom
      name: domain_check             # guardrails/domain_check.py
      mode: block
      config:
        fail_run: true               # mark blocked runs as 'failed' instead of 'completed'

  output:
    - type: moderation
      mode: block
    - type: pii_leakage
      mode: redact                   # block | warn | redact
      config:
        entities: [ssn, credit_card, api_key]
    - type: system_prompt_leakage
      mode: block
    - type: data_exfiltration
      mode: block
    - type: relevance
      mode: warn
    - type: regex
      mode: block
      config:
        patterns:
          - pattern: "internal-only"
            message: "Output contains internal-only content"
```

### Modes

- `block` — stop the run and return a rejection message.
- `warn` — log it and continue.
- `redact` — replace each detected entity with a type-specific placeholder (`[EMAIL_REDACTED]`, `[SSN_REDACTED]`, `[CREDIT_CARD_REDACTED]`, …) and continue. **Only valid for `pii` and `pii_leakage`** — all other types only support `block` and `warn`.

Every guardrail evaluation is recorded as a trace span. Violations appear in the trace timeline with the rule name, status, direction, provider, and detection details. A guardrail span has status `passed`, `blocked`, `warned`, or `redacted` and direction `input` or `output`.

### Built-in types

**Input**: `prompt_injection`, `pii`, `moderation`, `topic_restriction`, `regex`, `custom`.

**Output**: `moderation`, `pii_leakage`, `system_prompt_leakage`, `data_exfiltration`, `relevance`, `regex`, `custom`.

`topic_restriction` and `relevance` make an additional LLM call per run, which adds latency to the perceived response time.

### Provider-backed guardrails

For production grade detection, swap the default classifier for a managed one with `config.provider`:

```yaml
guardrails:
  input:
    - type: prompt_injection
      mode: block
      config:
        provider: lakera         # default | lakera (for prompt_injection)
```

Available providers vary by type: `moderation` supports `openai` and `perspective`; `prompt_injection` supports `lakera`. The deployed runtime also accepts `openai` and `perspective` for `pii_leakage`, although the current provider table only documents them for moderation. `topic_restriction` and `relevance` are LLM classifiers: choose their model with `config.model`, which defaults to the agent model, rather than `config.provider`. Input `pii`, `regex`, `system_prompt_leakage`, and `data_exfiltration` use Connic's local checks.

### Common config fields

- `mode`: `block` | `warn` | `redact` (per-type, see above).
- `rejection_message`: string shown to the caller when `mode: block` fires.
- `fail_run`: `true` to mark a blocked run as `status: failed` instead of `completed`. Default `false`.
- `off_topic_message` (topic_restriction only): the canned response.

### Type-specific config

| Type | Direction | Important `config` fields |
| --- | --- | --- |
| `prompt_injection` | input | `sensitivity: low \| medium \| high` (default `medium`); optional `provider: lakera` |
| `pii` | input | `entities`: any of `email`, `phone`, `ssn`, `credit_card`, `iban`, `ip_address`, `api_key` (default all) |
| `moderation` | input/output | `categories`; external-provider `threshold` (default `0.7`); optional `provider: openai \| perspective` |
| `topic_restriction` | input | `allowed_topics` or `blocked_topics`; `off_topic_message`; optional `model` (default agent model) |
| `regex` | input/output | `patterns`: a list of `{pattern, message}` objects |
| `pii_leakage` | output | `entities` (default `ssn`, `credit_card`, `api_key`) |
| `system_prompt_leakage` | output | `similarity_threshold` from 0 to 1 (default `0.6`) |
| `relevance` | output | optional relevance `context`; optional `model` (default agent model) |
| `data_exfiltration` | output | `allowed_domains` (default none, so every external domain is flagged) |

### Blocking and the `after` middleware

When an **input** guardrail blocks, the agent and `middleware/<agent>.py::before` are skipped and the rejection message is returned. The `after` middleware still runs on that rejection message by default — so it post-processes every response, blocked or not. To skip it on a block (matching how `before` and output guardrails are skipped), set the top-level `run_after_on_block` key under `guardrails:`:

```yaml
guardrails:
  run_after_on_block: false   # default true — skip `after` middleware when an input guardrail blocks
  input:
    - type: topic_restriction
      mode: block
      config:
        allowed_topics: [support, billing]
```

`run_after_on_block` is a top-level guardrails key (a sibling of `input`/`output`), not a per-rule `config` field.

### Custom guardrails

The filename must match `name:` exactly.

```yaml
guardrails:
  input:
    - type: custom
      name: domain_check          # guardrails/domain_check.py
      mode: block
```

```python
# guardrails/domain_check.py
from connic import GuardrailResult

ALLOWED_DOMAINS = ("acme.com", "example.com")

async def check(content: str, context: dict) -> GuardrailResult:
    if not any(d in content for d in ALLOWED_DOMAINS):
        return GuardrailResult(passed=False, message="Off-topic.")
    return GuardrailResult(passed=True)
```

The `check` function can be synchronous or asynchronous, and its signature is **exactly** `(content: str, context: dict)` — no `config` argument is passed in. `GuardrailResult` fields are `passed: bool`, `message: Optional[str]`, `details: Optional[dict]`. For output guardrails, `content` is the agent's response text. For redaction, use the built-in `pii`/`pii_leakage` types (they handle replacement internally) — there's no custom redact path.

`print`, `sys.stderr`, and stdlib `logging` output is captured under `guardrail.<name>` in Logs and the run detail. If a custom guardrail raises, Connic logs the traceback under that source and handles it as a guardrail violation according to the rule's mode: `warn` continues processing, while `block` stops processing and returns the configured rejection message.

## MCP servers

Connect external MCP (Model Context Protocol) servers to expose their tools to the agent. Tools from a configured server are **auto-loaded** into the agent — you do **not** list them again under `tools:`.

`mcp_servers:` uses Streamable HTTP, negotiates MCP `2026-07-28`, and falls back to legacy revisions without configuration changes.

```yaml
mcp_servers:
  - name: docs
    url: https://mcp.context7.com/mcp
    headers:
      Authorization: "Bearer ${MCP_TOKEN}"      # ${VAR} resolves at deploy time from env vars
      X-User-Id: "${context.user_id}"           # ${context.<path>} resolves per run from the run context
      X-Customer: "${context.customer.id}"      # dotted paths into nested dicts work
    tools: [read_file, list_directory]          # optional filter — limits which tools are exposed
    discoverable: false                          # true = LLM searches for tools on demand

  - name: internal
    url: http://mcp.internal:8080/mcp
    bridge: ${INTERNAL_BRIDGE_ID}                # tunnel through Connic Bridge for private endpoints
```

The optional `tools:` field **inside** the `mcp_servers[]` block filters which of the server's tools the agent sees. There is no `api:` prefix for MCP tools — that prefix is reserved for API-spec tools (next section). MCP tools appear in the agent's tool list under their own names.

`discoverable: true` defers loading — the LLM searches for matching tools by natural-language query at runtime instead of seeing them all upfront. Use it when a server exposes hundreds of tools.

### Header interpolation: deploy-time vs per-run

Two substitution syntaxes are supported inside `headers:` values. They run in this order and don't conflict:

| Syntax | Resolved | Source | Typical use |
| --- | --- | --- | --- |
| `${VAR}` | Once, at deploy time | Environment variables (Project Settings → Variables) | Service-level secrets that are the same for every run — bearer tokens, API keys |
| `${context.<dotted.path>}` | On every run | The run's `context` dict (populated by `middleware/<agent>.py::before`) | Per-user identity propagation — `X-User-Id`, `X-Customer`, anything else that must change per invocation |

When any header on a server uses `${context.*}`, the MCP toolset for that server is rebuilt per run so the substituted values stay fresh.

**Rules and gotchas for `${context.<path>}`:**

- The path is a dotted lookup into the per-run `context` dict — `${context.user_id}`, `${context.customer.id}`, etc. Only dict keys, no arbitrary attribute access.
- Keys starting with `_` are reserved and not interpolable (paths like `${context._system}` are rejected).
- Resolution is **strict**: a missing key raises `KeyError`, a `None` value raises `ValueError`, and the run fails. This is deliberate — identity propagation that silently falls back to an unauthenticated call would be worse than a hard failure.
- `${context.*}` only works in `headers:`. The `url:` field, the `bridge:` field, and the `tools:` list resolve `${VAR}` env-vars only.
- Populate the referenced keys in `middleware/<agent>.py::before` (the user-auth pattern in [tools-and-python.md](tools-and-python.md#end-user-authentication-and-per-run-permissions) shows the full flow). Tools that run before middleware can't see them.

This is the right way to forward the **authenticated end user's identity** to a downstream MCP server that needs to know who is acting (compliance servers, per-tenant data servers, anything that gates on user). Connic itself doesn't propagate user identity for you; you set the headers, and you set what goes into them in middleware.

Notes:

- Bridge IDs come from **Project Settings → Bridge** and are how you reach private MCP servers running inside your network.
- The server's host and port must be present in the Bridge agent's `ALLOWED_HOSTS`.
- A single agent can declare up to 50 MCP servers.
- MCP is supported only on LLM agents. External servers must use remote Streamable HTTP; stdio is not supported. Connic consumes tools only, not standalone resources, prompts, subscriptions, Tasks, or MCP Apps extensions.
- Authentication is via static or interpolated headers. Automatic MCP OAuth discovery and interactive authorization are not supported.
- If a server is unavailable at startup, the agent starts without that server's tools. A later MCP tool error is traced and returned to the LLM in the current run; Connic does not replay the agent or blindly retry a tool that may have produced side effects.
- Tool calls are traced. MCP tool annotations, structured results, and UI resource metadata attached to tools are preserved across supported protocol revisions.

## API spec tools

Import an OpenAPI 3.x spec and the SDK auto-generates one tool per operation. Configure in **Dashboard → Composer Settings → Add API Spec**.

Steps:

1. Set a name used as the prefix, e.g. `stripe`: at most 64 characters, starting with a lowercase letter and containing only lowercase letters, digits, and underscores.
2. Upload a JSON/YAML spec or supply a URL (refresh on demand).
3. Override the base URL if needed.
4. Configure auth: Bearer token, API Key (header), or Basic. Credentials are stored securely and injected at request time.

Tool naming:

- With `operationId`: converted to snake_case with the action verb at the end (`createCharge` → `charge_create`). If no verb is detected, the HTTP method is appended.
- Without: derived from path + verb (`GET /users` → `users_get`, `GET /users/{id}` → `users_by_id_get`).
- Common prefixes like `/api/v1` and `/v2` are stripped.

Reference in agent YAML:

```yaml
tools:
  - api:stripe.charge_create
  - api:stripe.*                # all operations
  - api:stripe.charge_*         # prefix wildcard
```

Each API spec lives in its own namespace — `api:stripe.charge_create` and `api:internal.charge_create` are independent tools.

After import, individual tools can be enabled or disabled and given custom names or descriptions. Disabled tools do not resolve through exact references or wildcards. Refreshing a URL-sourced spec matches existing tools by HTTP method and path, so custom names, descriptions, and enabled states survive while new endpoints are added and removed endpoints disappear. Any spec, tool, auth, or refresh change requires an agent redeploy before it takes effect.

Tool hooks in `hooks/<agent>.py` fire for local custom tools, predefined Connic tools, and `api:` tools. They do not fire for tools served by external MCP servers.

## When to use which

- **Output schemas** — when downstream code parses the agent's response. Replaces brittle prompt-engineering for JSON.
- **Guardrails** — when you need policy enforcement (PII, off-topic, prompt injection) at run boundaries. Cheaper and more reliable than asking the LLM to police itself in the prompt.
- **MCP servers** — when the tools you need already exist as an MCP server (Context7, your internal tools, third-party MCP catalogs).
- **API spec tools** — when you want every endpoint of an existing REST API exposed without writing wrapper functions.
- **Custom Python tools** — when you need project-specific logic, want to combine multiple API calls, or need to bake in defaults the LLM shouldn't have to think about.
