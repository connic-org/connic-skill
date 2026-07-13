# Agent YAML reference

Every agent is one YAML file in `agents/`. There are three agent types: `llm` (default), `sequential`, and `tool`.

## LLM agent — full schema

Required fields: `version`, `name`, `description`. For LLM agents, also `model`. Everything else is optional with defaults noted inline.

```yaml
version: "1.0"
name: support-assistant            # kebab-case; filename should match but isn't required to
type: llm                          # default; can be omitted
model: gemini/gemini-2.5-pro       # provider/model-name
fallback_model: gemini/gemini-2.5-flash   # used on primary provider failure
description: "Customer support agent with billing access"   # REQUIRED

system_prompt: |
  You are a concise support agent.
  User: {user_id} on {tier} tier.   # {var} resolves from `context`; truthy check via `{var}` substitution
  Use tools when helpful.

temperature: 0.1                   # 0 = deterministic, higher = more random
timeout: 45                        # seconds for one run; minimum 5; no default
max_iterations: 8                  # cap on LLM loop steps
max_concurrent_runs: 20            # default 1; cap on parallel runs of this agent
reasoning_effort: medium           # auto (default) | off | minimal | low | medium | high | xhigh
# reasoning_budget: 4096           # DEPRECATED; legacy Claude 3.7/Sonnet 4 and Gemini 2.5 only; use reasoning_effort

tools:                              # max 100 tools per agent
  - billing.lookup_invoice          # tools/billing.py::lookup_invoice
  - math.calculator.*               # all public funcs in tools/math/calculator.py
  - search.web_*                    # prefix wildcard (fnmatch)
  - query_knowledge                 # predefined Connic tool
  - trigger_agent                   # predefined: call another agent
  - premium.refund: context.tier == 'pro'   # conditional tool — context.* or input.*

discoverable_tools:                 # indexed for on-demand search, not loaded upfront
  - rare_tools.*                    # auto-injects `search_tools` + `use_tool` into the agent

output_schema: invoice              # schemas/invoice.json

retry_options:
  attempts: 5                       # max 10
  max_delay: 60                     # max 300 seconds (capped exponential backoff)
  rerun_middleware: true            # re-run before() on each retry

guardrails:
  run_after_on_block: true          # default; set false to skip the `after` middleware when an input guardrail blocks
  input:
    - type: prompt_injection
      mode: block                   # block | warn (redact only valid for pii / pii_leakage)
    - type: pii
      mode: redact
      config:
        entities: [email, phone, ssn]
  output:
    - type: system_prompt_leakage
      mode: block

mcp_servers:                        # max 50 servers per agent
  - name: docs
    url: https://mcp.context7.com/mcp
    headers:
      Authorization: "Bearer ${GITHUB_TOKEN}"   # ${VAR} = env var (resolved at deploy time)
      X-User-Id: "${context.user_id}"           # ${context.*} = per-run value from middleware
    tools: [read_file]              # optional filter; omit for all of the server's tools
    discoverable: false
    bridge: ${INTERNAL_BRIDGE_ID}   # optional Connic Bridge id for private MCP endpoints

session:
  key: context.chat_id              # dot-path into context or input; min ttl 60s
  ttl: 86400                        # seconds (omit = never expire)

context_compression:                # LLM agents only; omitted = compression off
  enabled: true                     # default true when block is present
  keep_recent_messages: 12          # default 8; recent messages kept verbatim
  session_history:                  # optional; off unless set
    interval: 4                     # compact stored session history every N runs
    keep_recent_runs: 1             # default 1; recent runs kept unsummarized
  max_prompt_tokens: 100000         # optional; early compression from model-reported prompt usage

concurrency:
  key: input.process_id             # serialize runs by this key
  on_conflict: queue                # queue | drop

approval:
  tools:
    - charge_customer
    - admin.delete: context.role == 'admin'    # conditions use param.* or context.*
    - refund: param.amount > 50                # `param.*` references the tool's arguments
  timeout: 3600                     # min 30s, max 604800s (7 days)
  message: "Confirm this action"
  on_rejection: fail                # fail (default) | continue (skip the tool and keep going)

# Access control for the predefined db_* and knowledge tools
database:
  prevent_delete: true              # block db_delete project-wide for this agent
  prevent_write: false              # block db_insert / db_update / db_upsert (per-collection rules also available)
knowledge:
  prevent_delete: true              # block delete_knowledge
  prevent_write: false              # block store_knowledge
```

## Sequential agent

Runs a pipeline of agents. Each agent's output becomes the next one's input.

```yaml
version: "1.0"
name: document-pipeline
type: sequential
description: "Validate → extract → format"
agents:
  - validate-input
  - extract-data
  - format-response
```

## Tool agent

Wraps a single Python tool with no LLM. The tool function **must declare a `payload` parameter and may declare `context`** — it's called as `func(payload=<dict>)`, plus `context=<dict>` if declared; any other parameter needs a default. Load/deploy validation rejects other signatures (the payload is never splatted into individual kwargs). A JSON-object trigger payload arrives as-is in `payload`, connector metadata keys included (a Kafka tombstone arrives as `{"message": None, "_kafka": ...}`); non-dict payloads are wrapped as `{"input": <value>}`. The run still appears in the dashboard with its inputs, outputs, and timing, but no model is invoked.

```yaml
version: "1.0"
name: tax-calculator
type: tool
description: "Compute tax for an order"
tool_name: calculator.calculate_tax
```

This is the right shape whenever you want the **connector and observability machinery** of a Connic agent without any reasoning step. Common cases:

- **Non-LLM Kafka / SQS / webhook consumers.** Link a Kafka (or SQS, webhook, S3, …) inbound connector to a `tool`-type agent and the message payload is passed straight into your Python function. No reasoning step is needed because the work is deterministic — but you still get auto-retries, run logs, judges, A/B variants, and everything else Connic gives a normal agent run.
- **Deterministic transforms and routing.** Parsers, validators, fan-out logic, ETL steps — anything where you know exactly what code should run.
- **Pre/post pipeline stages in a `sequential` agent.** Use tool agents as the deterministic stages around an LLM stage in a sequential pipeline.

When you need *some* deterministic logic alongside an LLM, you don't have to drop to a tool agent — you can also keep the LLM agent and put the deterministic work in `middleware/<agent>.py::before` (or `after`). Pick the tool-agent shape when there's *no* LLM step at all; pick the LLM-agent-with-middleware shape when the LLM still has something to do.

## Models

Format: `provider/model-name`. Examples:

- `openai/gpt-4o`, `openai/gpt-5.2`
- `gemini/gemini-2.5-pro`, `gemini/gemini-2.5-flash`
- `anthropic/claude-opus-4-7`, `anthropic/claude-sonnet-4-6`
- `azure/<deployment-name>` (requires Base URL + API version in dashboard)
- `bedrock/us.anthropic.claude-opus-4-7-v1:0`
- `vertex_ai/gemini-2.5-pro`
- `openrouter/anthropic/claude-sonnet-4.5`
- Custom prefix configured in **Project Settings → Models**: `<prefix>/<model-name>`

Don't guess model IDs — if a user wants a model not listed here, check the dashboard's model selector or the live docs.

## `tools:` list — patterns

```yaml
tools:
  - billing.lookup_invoice         # exact: tools/billing.py::lookup_invoice
  - billing.*                      # all public funcs in billing.py
  - support.search_*               # fnmatch wildcard
  - math.calculator.add            # nested: tools/math/calculator.py::add
  - api:stripe.charges_create      # API-spec tool (see guardrails-schemas-mcp.md)
  - api:stripe.*                   # all tools from an API spec
  - query_knowledge                # predefined tool (no module prefix)
  - admin.nuke: context.role == 'admin'      # conditional — only available when expr is true
  - web_search: input.internet_enabled       # conditional on payload field
  - alerts.page: context.urgent              # truthy check on a context value
```

**Tool conditions** use Python syntax with `context.*` and `input.*` accessors (validated at deploy time with `ast.parse` — names aren't resolved, only the syntax). Operators: `==`, `!=`, `>`, `<`, `>=`, `<=`, `and`, `or`, `not`, `in`. A bare `context.foo` (no comparator) is a truthy check.

**Approval conditions** are a different scope: they use `param.*` (the *tool arguments* about to be passed) and `context.*`. They do **not** support `input.*`. Don't confuse the two.

A tool cannot appear in both the unconditional list and a conditional entry — no duplicates.

### Full predefined-tool list

Reference by bare name in `tools:`:

`trigger_agent`, `trigger_agent_at`, `query_knowledge`, `store_knowledge`, `delete_knowledge`, `kb_list_namespaces`, `web_search`, `web_read_page`, `db_find`, `db_insert`, `db_update`, `db_upsert`, `db_delete`, `db_count`, `db_list_collections`.

See [predefined-tools.md](predefined-tools.md) for full signatures. Per [SKILL.md](../SKILL.md#best-practices-apply-these-by-default) best practice 1, prefer wrapping these in purpose-driven custom tools rather than handing them to the LLM raw.

### A/B test variants

The loader auto-recognises agent files named `{base-agent}-test-{variant}.yaml` (e.g. `support-assistant-test-shorter-prompt.yaml`) as A/B test variants of the base agent. See the dashboard's A/B Testing page for traffic split configuration.

## `system_prompt` variable interpolation

`{var_name}` is replaced with `context[var_name]` (set by middleware) before the prompt is sent. Unmatched placeholders pass through literally as `{var_name}`.

```yaml
system_prompt: |
  Assisting user {user_id} on the {tier} plan.
```

```python
# middleware/<agent>.py
async def before(content, context):
    context["user_id"] = "u_123"
    context["tier"] = "pro"
    return content
```

## Sessions

Persist conversation state across runs for the same key.

```yaml
session:
  key: context.chat_id    # dot-path; must resolve to non-empty value
  ttl: 86400              # seconds; omit = never expire
```

Each unique value of the key gets its own conversation history. Useful for chat-style agents triggered by webhooks where each user keeps state.

For long-running LLM sessions, add `context_compression` to keep prompts within the model context window:

```yaml
context_compression:
  enabled: true
  keep_recent_messages: 12
  max_prompt_tokens: 100000
```

Compression is off unless the block is configured. Once configured, provider context-window errors trigger compression and an automatic retry. `max_prompt_tokens` is optional and compresses earlier using prompt usage reported by prior model calls. `keep_recent_messages` controls how many recent messages stay verbatim. `session_history.interval` is optional and only needed if stored session history should be compacted between runs on a fixed cadence. Lower values compact older history more often. `context_compression` is only valid on `type: llm` agents.

Active sessions can be viewed and deleted in the dashboard under **Storage > Sessions**. Sessions are scoped per environment.

The key has to resolve to a non-empty value before the run starts, so `context.chat_id` needs to be populated in middleware (or come from `input.*` directly). Pull it from the raw connector payload — don't try to parse it out of the user's text message:

```python
# middleware/<agent>.py
async def before(content, context):
    payload = context.get("payload", {})
    context["chat_id"] = payload.get("chat_id")   # from the webhook body / query string
    return content
```

## Concurrency

Limit how many runs of this agent execute simultaneously for a given key.

```yaml
concurrency:
  key: input.tenant_id
  on_conflict: queue      # queue = wait; drop = discard the new run
```

`max_concurrent_runs` is a hard cap on total parallelism; `concurrency.key` adds per-key serialization on top.

## Approvals

Gate specific tool calls behind a human decision in the dashboard.

```yaml
approval:
  tools:
    - charge_customer
    - admin.delete: context.role == 'admin'    # condition: context.* or param.*
    - refund: param.amount > 50                # gate only when the argument exceeds threshold
  timeout: 3600           # min 30s, max 604800s
  message: "Approve this action"
  on_rejection: fail      # fail = stop run (default); continue = skip the tool and keep going
```

The run pauses at the gated tool call until a reviewer approves or rejects in **Dashboard → Approvals**. Approval conditions use `param.*` (the tool's arguments) and `context.*`; `input.*` is **not** valid here (unlike tool conditions in `tools:`).

## Cascading defaults with `_defaults.yaml`

Drop a `_defaults.yaml` into any directory under `agents/` to share configuration with every agent at that directory level and below. Use it to factor out the boring repetition — the same `model`, the same `guardrails`, a common set of `tools`, the same `database.collections` — across families of agents.

```
agents/
├── _defaults.yaml              # applies to every agent in the project
├── process/
│   ├── _defaults.yaml          # adds/overrides for everything under process/
│   ├── ingest/
│   │   └── foo.yaml
│   └── enrich/
│       └── bar.yaml
└── support-assistant.yaml
```

The loader collects the chain of `_defaults.yaml` files from `agents/` down to the agent's directory (shallowest first) and merges them; the agent's own YAML is applied last and wins on conflict.

### What can live in `_defaults.yaml`

The same fields as a normal agent YAML, but partial — only set what you want to share. Two exceptions:

- `name` and `description` are **forbidden** in defaults (they're per-agent identity; sharing them would either be useless or collide).
- `version` is allowed in defaults so you can pin `"1.0"` once project-wide. The agent file must still set it too (along with `name` and `description`).

The linter rejects defaults files that contain `name` or `description`, pointing at the offending file.

### Merge rules

Applied recursively as the chain is folded together:

- **Scalars** (`model`, `temperature`, `system_prompt`, `timeout`, `reasoning_effort`, …) — deeper layer replaces shallower; the agent file replaces all.
- **Dicts** (`database`, `knowledge`, `retry_options`, `approval`, `session`, `concurrency`, `guardrails`, `database.collections`, `knowledge.namespaces`, …) — recursive deep merge, per-key. Defaults can supply some collections; an agent can add more without losing the inherited ones.
- **Lists** — concat with dedup, so children **add to** rather than replace inherited lists:
  - `tools`, `discoverable_tools`, `approval.tools` — dedup by tool ref. An agent re-declaring an inherited tool (e.g. with a different condition) overrides the inherited entry.
  - `mcp_servers` — dedup by server `name`. An agent's full server config replaces the inherited one on collision.
  - `guardrails.input`, `guardrails.output` — dedup by rule `name` if present; otherwise appended.
  - Sequential `agents:` — dedup by string.

Order in the final list: inherited entries first (root → deepest), then the agent's own, minus anything the agent re-declared.

### Example

`agents/_defaults.yaml`:

```yaml
version: "1.0"
type: llm
model: anthropic/claude-sonnet-4-6
temperature: 0
guardrails:
  input:
    - type: prompt_injection
      mode: block
  output:
    - type: system_prompt_leakage
      mode: block
tools:
  - audit.log_event
```

`agents/billing/refund-agent.yaml`:

```yaml
version: "1.0"
name: refund-agent
description: "Issues refunds against the billing system."
system_prompt: "Refund only valid charges. Use the tools."
tools:
  - billing.lookup_charge
  - billing.issue_refund
approval:
  tools:
    - billing.issue_refund: param.amount > 50
  timeout: 3600
```

Effective config for `refund-agent`: `model: anthropic/claude-sonnet-4-6`, `temperature: 0`, the two inherited guardrails, and `tools: [audit.log_event, billing.lookup_charge, billing.issue_refund]`. The agent didn't redeclare `audit.log_event`, so it's inherited as-is.

### Tips

- Each `_defaults.yaml` is parsed once per loader run, so adding shared config doesn't hurt cold-load time even with many agents.
- Use `connic lint` to see the final shape if you're unsure what got merged — load errors point at the file whose values caused the rejection.
- Test variants (`<base>-test-<name>.yaml`) inherit the same defaults as their base, since they live in the same directory.

## Common mistakes

- Writing `tools/billing.py` and referencing `billing.lookup_invoice` but the function is named `lookupInvoice` — case and snake_case matter.
- Using `tools: lookup_invoice` (string) instead of `tools: [lookup_invoice]` (list).
- Omitting `version`, `name`, `description`, or `model` (LLM agents) — the linter rejects the file. `description` is required and often forgotten. None of `name`, `description`, or `version` can be supplied by a `_defaults.yaml`; `name` and `description` are forbidden there entirely.
- Setting `session.key` to a literal string instead of a `context.*` / `input.*` path.
- Listing the same tool both unconditionally and conditionally — not allowed.
- Using `input.*` in an `approval.tools` condition — only `param.*` and `context.*` are valid there. Conversely, using `param.*` in a `tools:` conditional — that's only valid in approvals.
