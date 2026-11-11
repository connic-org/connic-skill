# Agent YAML reference

Every agent is one YAML file in `agents/`. There are three agent types: `llm` (default), `sequential`, and `tool`.

## Text LLM agent — full schema

Required fields: `version`, `name`, `description`. LLM agents also require `model` and `system_prompt`; sequential agents require `agents`; tool agents require `tool_name`. Everything else is optional with defaults noted inline.

This YAML configures a connic agent, separate from the plugin's bundled connic MCP connection.

```yaml
version: "1.0"
name: support-assistant            # kebab-case; filename should match but isn't required to
type: llm                          # default; can be omitted
model: connic/glm-5.2              # exact connic catalog ID; no provider key
fallback_model: anthropic/claude-sonnet-4-6   # BYOK fallback; uses Project credentials
description: "Customer support agent with billing access"   # REQUIRED

system_prompt: |
  You are a concise support agent.
  User: {user_id} on {tier} tier.   # interpolated from context; unmatched placeholders stay literal
  Use tools when helpful.

temperature: 0.1                   # default 1.0; range 0.0–2.0
timeout: 45                        # seconds for one run; minimum 5; no default; capped by the plan's run timeout (connic.co/pricing)
max_iterations: 8                  # default 100; minimum 1
max_concurrent_runs: 20            # default 1; minimum 1
reasoning_effort: medium           # auto is the default; supported overrides are model-specific
reasoning_budget: 4096             # optional; only for models that accept a raw token budget

tools:                              # max 200 combined with generated approval.inputs tools; MCP server tools and discoverable_tools don't count
  - billing.lookup_invoice          # tools/billing.py::lookup_invoice
  - math.calculator.*               # all public funcs in tools/math/calculator.py
  - search.web_*                    # prefix wildcard (fnmatch)
  - retrieval_query                # predefined connic tool
  - trigger_agent                   # predefined: call another agent
  - premium.refund: context.tier == 'pro'   # conditional tool — context.* or input.*

discoverable_tools:                 # indexed for on-demand search, not loaded upfront
  - rare_tools.*                    # auto-injects `search_tools` + `use_tool` into the agent

output_schema: invoice              # schemas/invoice.json

retry_options:
  attempts: 3                       # attempts for the model being retried/operation; max 10
  attempt_timeout: 120              # positive seconds per full LLM request/stream; capped by remaining run time
  initial_delay: 10                 # initial delay; range 0–300 seconds
  max_delay: 30                     # generated backoff cap; range 1–300 seconds
  rerun_middleware: false           # tool/sequential operation retries only; ignored for LLM agents

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
      X-Tenant-Id: "${TENANT_ID}"               # ${VAR} = Project variable (resolved at deploy time)
      X-User-Id: "${context.user_id}"           # ${context.*} = per-run value from middleware
    tools: [read_file]              # optional filter; omit for all of the server's tools
    discoverable: false
    bridge: ${INTERNAL_BRIDGE_ID}   # optional connic bridge id for private MCP endpoints
    condition: context.user_id != null  # optional; connect only when true (conditional-tool grammar)
    optional: false                 # true = run continues without this server if connecting fails
    oauth:                          # optional per-user sign-in; no Authorization header allowed with it
      app: docs                     # OAuth app name in the environment (Settings → OAuth & Passkeys)
      user: context.user_id         # context.* path holding the end user's ID
      connect_tool: false           # true = connect_<server> tool returns a sign-in link when not connected

browser:
  passkeys: [insurer]               # max 10 passkey names from Settings → OAuth & Passkeys; deploy fails if one is missing

session:                           # true = one shared session; false/null/omitted = off
  key: context.chat_id              # optional context.* or input.* path; omitted/null = shared
  ttl: 86400                        # inactivity seconds, min 60; omitted/null = no expiry
  history: true                     # retain conversation history across runs
  files: true                       # false = files from earlier runs become short notes in the history
  browser: true                     # retain browser cookies and local storage across runs

context_compression:                # LLM agents only; omitted = compression off
  enabled: true                     # default true when block is present
  model: connic/qwen3.6-35b-a3b     # optional; compression summaries only; default = agent model
  keep_recent_messages: 12          # default 8; recent messages kept verbatim
  session_history:                  # optional; off unless set
    interval: 4                     # compact stored session history every N runs
    keep_recent_runs: 1             # default 1; recent runs kept unsummarized
  max_prompt_tokens: 100000         # optional; early compression from model-reported prompt usage

concurrency:
  key: process_id                   # dot-path into the raw trigger payload
  on_conflict: queue                # queue | drop | coalesce

approval:
  tools:
    - charge_customer
    - admin.delete: context.role == 'admin'    # conditions use param.* or context.*
    - refund: param.amount > 50                # `param.*` references the tool's arguments
  inputs:                                      # LLM agents only; generated human-input tools
    - get_mfa:
        prompt: Use this tool when authentication requires an MFA code.
        label: MFA code
        sensitive: true
        params:
          - reason: str
          - account_email: str
  timeout: 3600                     # min 30s, max 604800s (7 days)
  message: "Confirm this action"
  on_rejection: fail                # fail (default) | continue (skip the tool and keep going)

# Access rules for the built-in db_*, retrieval_* and file_* tools, the same format as apps/<key>.yaml.
# Omit access for unrestricted access. Once set, anything without a rule is denied.
lookups:                            # optional, needs access
  member:
    collection: members
    where: doc.email == user.email  # where may use doc, user, context, session
  teams:
    collection: memberships
    where: doc.user_id == user.id
    values: team_id                 # list of values from every match, max 1000 documents
access:
  database:                         # read, insert, update, delete, set, fields
    "*":                            # every collection without its own entry (quote the key)
      read: true
    orders:                         # a named entry wins over "*", even an empty one
      read: user == null or doc.customer_id == member.customer_id
      insert: true
      update: user == null or doc.customer_id == member.customer_id
      set:
        source: '"support-agent"'          # written on insert, overriding the supplied value
        updated_by:
          value: '"support-agent"'
          writes: [insert, update]
      fields:
        internal_notes: user == null       # field hidden from visitor-driven runs
  files:                            # read, write, update, delete, set; also covers file_share_link,
    uploads:                        # retrieval_store(file_path=...), retrieval_query share links, browser up/downloads
      read: file.session_id == session.id  # replaces session_only; requires session
      write: true
      update: file.session_id == session.id
    templates:
      read: true
  retrieval:                        # search, store, delete; a search covers the namespaces below it
    policies:
      search: true
    "*":
      search: true                  # needed for retrieval_query without a namespace
```

Agent rules see `user` (the app visitor the run acts for: `id`, `email`, `name`, `groups`, `claims`, `signed_in`; also in agents started with `trigger_agent`; `null` for runs from cron, webhooks, the dashboard), `context` (the run context), `session` (`session.id`, `null` without a session), the lookups, and the record (`doc`, `new`, `file`). Rules on `user` deny runs without a visitor; write `user == null or ...` to let them through. `new` in a database `update` rule is the document after the change: sent fields plus stored values for the rest, used only through its fields (`new.status`), and two fields of `new` can't be compared. Rules apply to every call of the built-in tools, also from custom tools through `connic.tools`. `connic lint` fails when an agent with `access` lists a built-in tool whose resource has no entries. Up to 100 entries per resource. Full rule reference: `https://connic.co/docs/v1/build/access-rules`

The older `database`, `retrieval` and `files` blocks (`collections`/`namespaces`/`folders`, `prevent_write`, `prevent_delete`, `session_only`) are deprecated. They still work, can't be combined with `access`, and `connic lint` prints the `access` block that replaces them. Write new agents with `access`.

## LLM retries and timeouts

For `type: llm`, every failed model request can use the configured retry or fallback path while time and attempts remain. The only retryability filters are `unsupported_file_type` and `unsupported_modality`, which are not retried on the same model but can still switch to the fallback. `attempts` includes the first ordinary request and defaults to three: without `fallback_model`, the primary gets that budget; with a fallback, the primary is tried once and the fallback gets up to `attempts` requests. The single context-compression repair described below is separate from these counts. connic never restarts the whole LLM agent.

`attempt_timeout` defaults to 120 seconds and accepts positive, finite seconds, including fractions. It caps each complete request or entire stream at the remaining run time, whichever is shorter. Receiving stream chunks does not restart the timer. Retries do not divide the remaining time into smaller budgets. Run expiry or cancellation stops all attempts; once text, reasoning, or tool-call output has been streamed, connic never retries, switches models, or repairs context for that request.

Switching to a fallback is immediate. Retrying the same model honors `Retry-After` when supplied, otherwise the configured backoff. These waits count against the run deadline; connic stops if a wait leaves no time for another attempt. Request retries preserve tool results already produced in the run, and tool failures are returned to the LLM instead of blindly re-invoked. `attempts` also controls the tool-failure reflection budget. Runs that switch models expose `context.fallback_model_used` and show the switch in the trace.

For `type: tool` and `type: sequential`, `attempts`, `initial_delay`, `max_delay`, and `rerun_middleware` control operation-level retries. `attempts` includes the first attempt; `attempt_timeout` is ignored.

## Voice LLM agent

connic voice is in beta. Add `voice_config` to an LLM agent that uses a native realtime audio model. An empty `voice_config: {}` enables voice with default settings.

```yaml
version: "1.0"
name: phone-assistant
type: llm
model: openai/gpt-realtime-2.1
description: "Phone support agent"
system_prompt: |
  Keep replies short and natural.
  Say what you are checking before calling a tool.
voice_config:
  voice: alloy
  language: en
  transcription_model: gpt-realtime-whisper
  greeting: "Hi! How can I help?"
  thinking_sound: true
  hang_up_allowed: true
  press_keys_allowed: true
  wait_on_hold_allowed: false
  turn_detection:
    silence_ms: 500
  interruptions:
    enabled: true
    min_speech_ms: 500
  idle_timeout_seconds: 60
tools:
  - support.lookup_account
```

All `voice_config` fields are optional except `interruptions.enabled` when the `interruptions` object is present. `thinking_sound`, `hang_up_allowed`, and `press_keys_allowed` default to `true`, `wait_on_hold_allowed` to `false`; omitting `interruptions` leaves interruptions enabled, and omitting `greeting` makes the agent wait for the other side to speak first, on incoming and outgoing calls. `silence_ms` defaults to 500 milliseconds and must be greater than zero. `interruptions.min_speech_ms` is how long the caller must speak while the agent talks before the agent stops; it defaults to 500 milliseconds and accepts 0 to 5000. `idle_timeout_seconds` must be greater than zero and counts only speech (the agent's and the other side's transcribed speech); running tools and waiting on hold keep the call open. `language` takes a code such as `de-DE`: it goes into the instructions, which set the spoken language, and to the provider's transcription.

For OpenAI, `transcription_model` selects the model that transcribes caller speech. For Azure, it names an existing transcription deployment in the same resource. Omit it to disable caller transcription for those providers. Gemini, Vertex AI, and `connic/gemini-3.8-live` provide native transcription without this setting.

Voice agents require `type: llm` and a voice model. `connic/gemini-3.8-live` needs no provider credentials and bills its audio and text tokens from Project credit; it takes Gemini voice names such as `Kore`. With provider credentials, use a model prefixed with `openai/`, `azure/`, `gemini/`, or `vertex_ai/`; the selected provider determines which realtime model and voice IDs are available. Other `connic/*` models don't support voice.

Set `reasoning_effort` to request a reasoning level. connic passes the setting to the provider, which rejects unsupported settings or values. Omit it or use `auto` for the model default.

Python tools, retrieval, MCP servers, conditional and discoverable tools, tool hooks, delegation, and middleware remain available. connic rejects input and output guardrails, `approval`, `output_schema`, `output_schema_dict`, and `fallback_model` because it streams voice audio immediately. Automated test suites do not run voice sessions; use `connic lint`, deploy the agent, and test it in a live conversation.

Calls longer than about 15 minutes on Gemini, Vertex AI, and `connic/gemini-3.8-live` need `context_compression`; without it, the provider ends the call. With it, the provider drops or summarizes the oldest part of the call and keeps the instructions. On voice agents `context_compression` takes only `enabled` and `max_prompt_tokens` (compress once the call reaches that many tokens); `model`, `keep_recent_messages`, and `session_history` fail validation. OpenAI and Azure drop the oldest turns once a call fills the context window; `max_prompt_tokens` sets how many tokens they keep.

Billing: a call is one run, billed as the run start plus runtime per second from the start of the run to the end of the call, including an outgoing call's ringing and time on hold. There is no voice fee. The model provider bills tokens when using customer-provided model credentials; the phone provider bills the call. connic stores no call audio; the transcript is the run output and appears in traces. Data location: Vertex AI runs in the project's saved location (`eu` = EU multi-region); Azure runs in the resource's region (EU region plus Data Zone deployment); OpenAI's endpoint and Gemini API keys have no region setting. Phone-side settings are in [connectors.md](connectors.md).

Use connic connectors to answer or place phone calls with a deployed voice agent. Configure the connector in the Dashboard or in `connectors/*.yaml`, not in agent YAML. Twilio Voice and Telnyx Voice use a saved Connection. See [connectors.md](connectors.md) for setup instructions.

An incoming call starts the run with `call_id` (the provider's call identifier), `from`, `to`, `provider` (the voice connector that received the call), and `caller_name` and `forwarded_from` when the provider sends them. Twilio and Telnyx also include `raw`, the provider's complete incoming-call request. What each provider sends is in [connectors.md](connectors.md). Middleware reads it from `context["payload"]` and tools through their `context` parameter, and the model receives it as the first message unless `before` middleware replaces it. To give callers different tools by identity, look the caller up in `before` middleware, store the result in `context`, and gate tools with conditional tools. An outgoing call keeps its run's own input: context has no call ID or phone numbers, and `context["connector_id"]` is the connector that started the run, not the voice connector, so put what the agent needs in the run input. One call is one run, so the agent's `timeout`, capped by the plan's run timeout (60 minutes on Pro), is the maximum call length. Each agent reply counts toward `max_iterations`; replies that only go back on hold don't.

To transfer a caller, raise `TransferCall("+4930123456789")` (from `connic`) in a tool or tool hook. Speech already playing finishes first; the number must be one of the voice connector's transfer numbers, otherwise the tool call returns an error. Keep any conditions for the transfer in the tool. The Playground shows the number instead of dialing it. If the transfer number doesn't answer, the call ends; the connector's fallback number covers only incoming calls in sync mode, never transfers or outgoing calls. The transfer ends the run normally, so `after` middleware still runs; the tool's trace step shows the number, but `context` does not record it. If `after` needs to know, set a key such as `context["transferred_to"]` in the tool before raising.

`voice_config.keypad` maps keys (`0`-`9`, `*`, `#`) to tool refs that are also listed in `tools`; quote the keys in YAML. A key press runs the tool without a model turn, at any point in the call, so the tool cannot have required parameters. A keypad tool that raises `TransferCall` stops current speech and transfers immediately.

```yaml
tools:
  - phone.transfer_to_operator
voice_config:
  greeting: "To reach an operator, press 0 at any time."
  keypad:
    "0": phone.transfer_to_operator
```

The built-in `press_keys` tool lets the agent press phone keys itself, such as in a phone menu: `keys` takes `0`-`9`, `*`, `#`, and `w` for a half-second pause, up to 64 (`1w2#`). The keys play on the call in order and the agent then waits for the menu; each press is recorded in the run and shown on the tool's trace step. It works the same on every voice connector; the Playground shows the keys instead of sending them. `press_keys_allowed: false` removes the tool.

With `wait_on_hold_allowed: true`, the agent gets the built-in `wait_on_hold` tool and calls it once the call is in a queue or on hold. While it waits, the model gets no audio (no tokens) and makes no replies, and waiting doesn't count toward `max_iterations` or `idle_timeout_seconds`. connic wakes the agent with the speech once speech is followed by a quiet line (a person greets and pauses; music and announcements keep playing) and doesn't wake it again for a recording it went back on hold after without speaking. An announcement followed by silence wakes it once, so tell the agent to call `wait_on_hold` again when music or a recording woke it. `max_hold_seconds` (1-3600, needs `wait_on_hold_allowed`) ends the call and fails the run once a wait lasts that long; a wait lasts until the agent speaks after waking.

```yaml
system_prompt: |
  You call suppliers to ask when an open order ships.
  In a phone menu, choose the option for order status with press_keys.
  When you are put on hold, call wait_on_hold. When a person answers,
  say who you are and which order you are calling about.
voice_config:
  language: de-DE
  wait_on_hold_allowed: true
  max_hold_seconds: 2400
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

Wraps a single custom Python tool under `tools/` with no LLM. Predefined tools, `api:` tools, and wildcard references cannot be a tool agent's body. The tool function **must declare a `payload` parameter and may declare `context`** — no other parameters are permitted, including defaulted parameters. connic calls it with the normalized payload dict and injects `context` when declared; it never expands payload keys into separate arguments. JSON objects arrive as-is, while decoded non-object values are wrapped as `{"input": <value>}`. In a sequential pipeline, the tool's JSON-compatible return value keeps its type for the next stage. The run appears in the dashboard with its input, output, and timing, but no model is invoked.

```yaml
version: "1.0"
name: tax-calculator
type: tool
description: "Compute tax for an order"
tool_name: calculator.calculate_tax
```

Use a tool agent when a connector or sequential pipeline should run deterministic Python without a reasoning step. Common cases:

- **Non-LLM Kafka / SQS / webhook consumers.** Link an inbound connector to a `tool`-type agent and its normalized payload is passed to the Python function. Runs retain logs, configured retries, and judges.
- **Deterministic transforms and routing.** Parsers, validators, fan-out logic, ETL steps — anything where you know exactly what code should run.
- **Pre/post pipeline stages in a `sequential` agent.** Use tool agents as the deterministic stages around an LLM stage in a sequential pipeline.

When you need *some* deterministic logic alongside an LLM, you don't have to drop to a tool agent — you can also keep the LLM agent and put the deterministic work in `middleware/<agent>.py::before` (or `after`). Pick the tool-agent shape when there's *no* LLM step at all; pick the LLM-agent-with-middleware shape when the LLM still has something to do.

## Models

Format: `provider/model-name`. Models can be connic-managed or BYOK:

- `connic/glm-5.2` — connic-managed, EU-hosted, billed against Project credit; no provider key
- `openai/gpt-4o`, `openai/gpt-5.2`
- `gemini/gemini-2.5-pro`, `gemini/gemini-2.5-flash`
- `anthropic/claude-opus-4-7`, `anthropic/claude-sonnet-4-6`
- `azure/<deployment-name>` (requires Base URL + API version in dashboard)
- `bedrock/us.anthropic.claude-opus-4-7-v1:0`
- `vertex_ai/gemini-2.5-pro`
- `openrouter/anthropic/claude-sonnet-4.5`
- Custom OpenAI-compatible prefix configured in **Project Settings**: `<prefix>/<model-name>`

Use exact `connic/*` IDs from the live [connic model catalog](https://connic.co/docs/v1/build/connic-models). For BYOK or custom providers, use an ID supported by the configured provider. Don't invent aliases or substitute another model.

For a custom OpenAI-compatible provider, configure a unique lowercase prefix, the API base URL, and an optional API key. The `connic` prefix is reserved for managed models.

An exact ID ending in `-fast` selects a latency-optimized catalog profile. Some models only have a Fast ID; where both exist, the profiles can differ in output quality, accepted `reasoning_effort` values, context size, or input modalities. Treat each exact ID as distinct and use the limits shown in the model catalog.

Managed and BYOK models can be paired as primary and fallback in either direction.

## `tools:` list — patterns

```yaml
tools:
  - billing.lookup_invoice         # exact: tools/billing.py::lookup_invoice
  - billing.*                      # all public funcs in billing.py
  - support.search_*               # fnmatch wildcard
  - math.calculator.add            # nested: tools/math/calculator.py::add
  - api:stripe.charge_create       # API-spec tool (see guardrails-schemas-mcp.md)
  - api:stripe.*                   # all tools from an API spec
  - retrieval_query               # predefined tool (no module prefix)
  - db_*                          # all predefined database tools
  - admin.nuke: context.role == 'admin'      # conditional — only available when expr is true
  - web_search: input.internet_enabled       # conditional on payload field
  - alerts.page: context.urgent              # truthy check on a context value
```

**Tool conditions** use connic's restricted expression language with `context.*` and `input.*` accessors. Comparisons are `==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `not in`, `is`, and `is not`; combine or group them with `and`, `or`, `not`, and parentheses. Dot access, subscripts, and string/number/boolean/null and list/tuple/set/object literals are allowed. A bare accessor is a truthy check. Syntax errors fail validation. During a run, a missing field, unsupported operation, or other evaluation error makes the condition false rather than exposing the tool. Expressions are limited to 2,000 characters; calls, imports, arithmetic, assignments, private attributes, and roots other than `context` and `input` are not allowed.

**Approval conditions** are a different scope: they use `param.*` (the *tool arguments* about to be passed) and `context.*`. They do **not** support `input.*`. Don't confuse the two.

A tool cannot appear in both the unconditional list and a conditional entry — no duplicates.

Every exact tool reference must resolve. A wildcard without a dot matches predefined tool names, so `db_*` expands to every predefined database tool. Every wildcard must match at least one eligible tool; a zero-match wildcard fails deployment rather than silently producing an empty set.

An agent can expose at most 200 tools. Each function a `tools` entry resolves to counts once: `billing.*` counts every sync or async function defined in `tools/billing.py` whose name does not start with `_`, but not functions imported into that file, and a predefined or `api:` wildcard counts every tool it matches (`web_browser_*` is 15). Generated `approval.inputs` tools count, and so do `search_tools` and `use_tool` once any tool is discoverable. `discoverable_tools`, MCP server tools, and agent-tool outbound connectors don't count. `connic lint` reports an agent over the limit, and `connic test`, `connic dev`, and deploys reject it. For larger sets, move rarely used tools to `discoverable_tools`.

Use `discoverable_tools` for tools the agent should find by natural-language search instead of loading upfront. It accepts the same exact references, wildcards, and conditions as `tools`. When any local or MCP tools are discoverable, connic provides `search_tools` and `use_tool` automatically; do not list them yourself. A function cannot appear in both `tools` and `discoverable_tools`, and exposed function names must be unique across both lists. A tool whose condition is false for the run doesn't show up in `search_tools` and can't be called. Tool hooks and `approval.tools` treat a discovered tool like one in `tools`; hooks see the tool's own name, not `use_tool`. `search_tools` matches the words of the query against tool names and descriptions, so descriptions must use the language the agent works in. Finding a tool costs an extra model step before the `use_tool` call; keep frequently used tools in `tools` and reserve discovery for large, rarely used sets.

### Full predefined-tool list

Reference by bare name in `tools:`:

`trigger_agent`, `trigger_agent_at`, `retrieval_query`, `retrieval_store`, `retrieval_delete`, `retrieval_list_namespaces`, `web_search`, `web_read_page`, `web_browser_open`, `web_browser_observe`, `web_browser_read`, `web_browser_act`, `web_browser_close`, `web_browser_screenshot`, `web_browser_mouse`, `web_browser_tabs`, `web_browser_dialog`, `web_browser_eval`, `web_browser_console`, `web_browser_network`, `web_browser_upload`, `web_browser_download`, `web_browser_pdf`, `db_find`, `db_insert`, `db_update`, `db_upsert`, `db_delete`, `db_count`, `db_list_collections`, `file_list`, `file_read`, `file_write`, `file_update`, `file_delete`, `file_share_link`.

See [predefined-tools.md](predefined-tools.md) for full signatures. Per [SKILL.md](../SKILL.md#best-practices-apply-these-by-default) best practice 1, prefer wrapping these in purpose-driven custom tools rather than handing them to the LLM raw.

### A/B test variants

Agent files named `{base-agent}-test-{variant}.yaml` (e.g. `support-assistant-test-shorter-prompt.yaml`) are A/B test variants of the base agent. See the dashboard's A/B Testing page for traffic split configuration.

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

`session: true` uses one shared session across the agent’s runs. An object with no `key` or `key: null` uses the same shared session. `false`, `null`, or omitting `session` disables persistence.

`history` and `browser` default to `true`. This example keeps cookies and local storage per chat while starting every run with fresh conversation history:

```yaml
session:
  key: context.chat_id
  ttl: 86400
  history: false
  browser: true
```

The optional `key` is a `context.*` or `input.*` dot path that can reach nested fields (`input.user.id`); each resolved value gets a separate session. `ttl` is an inactivity timeout in seconds, at least 60; omitted or `null` means no expiry. `browser: false` disables profile persistence while browser tools remain available with fresh state. `files: false` keeps files from earlier runs out of the reused history, so the model does not receive them again with every new message; each one becomes a short note with its file type, and files sent in the current run still reach the model.

Runs sharing a session are not serialized: if two overlap, the run that writes second fails. When runs for the same session can overlap, add `concurrency.key` on the same payload field (e.g. `concurrency.key: conversation_id` with `session.key: input.conversation_id`).

For long-running LLM sessions, add `context_compression` to keep prompts within the model context window:

```yaml
context_compression:
  enabled: true
  model: connic/qwen3.6-35b-a3b
  keep_recent_messages: 12
  max_prompt_tokens: 100000
```

Compression is off unless the block is configured. When enabled, a context-window error permits one reactive compression recovery per logical model call, shared across primary and fallback. If compression changes the request, connic makes one extra request to the same active model before considering fallback. That repair request is outside the ordinary `attempts` budget and uses the same per-request timeout and remaining run deadline. It does not reset counters or return from fallback to primary. Failed or unchanged compression uses up the recovery allowance, then ordinary retry or fallback continues with the last usable request. A second context overflow does not trigger another repair. No recovery occurs after streaming output begins.

`model` is optional and selects only the context and stored-history summary model; when omitted, summaries use the agent model. Summary requests have their own bounded attempt sequence, share the run deadline and `attempt_timeout`, and cannot recursively compress. The repaired agent request stays on whichever primary or fallback model overflowed. `max_prompt_tokens` is optional and compresses earlier using prompt usage reported by prior model calls. `keep_recent_messages` controls how many recent messages stay verbatim. `session_history.interval` is optional and only needed if stored session history should be compacted between runs on a fixed cadence. Lower values compact older history more often. Every `interval`-th run ends by summarizing older stored history, keeping the last `keep_recent_runs` runs verbatim. Without it, each run loads the full stored history and compression only shortens requests. Stored history has no total size or run-count limit; each stored message can be up to 5 MB. `context_compression` is only valid on `type: llm` agents.

Active sessions can be viewed and deleted in the dashboard under **Storage > Sessions**. Sessions are scoped per environment.

A configured key path has to resolve to a non-empty value before the run starts, so `context.chat_id` needs to be populated in middleware (or come from `input.*` directly). Pull it from the raw connector payload — don't try to parse it out of the user's text message:

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
  key: data.tenant_id
  on_conflict: queue      # queue = wait; drop = discard the new run; coalesce = at most one waiting run, newest event wins
```

`concurrency.key` is a dot-path into the **raw trigger payload**; unlike `session.key`, it does not use an `input.` prefix. If the path does not exist, per-key enforcement is skipped for that run. `max_concurrent_runs` is a hard cap on total parallelism: further runs wait in arrival order with no length cap or wait timeout, are not dropped, and their wait does not count against `timeout`. Incoming phone calls use the wait limit in [Voice call settings](connectors.md#voice-call-settings). `concurrency.key` adds per-key serialization on top, and runs waiting for their key don't take a slot. Concurrency rules are supported on LLM and tool agents, not sequential agents.

Use `coalesce` when every event means "re-check the current state" (status updates, webhook bursts): events that arrive during a run collapse into one follow-up run with the latest payload, and replaced runs end as cancelled with `superseded_by` in their context. Prefer it over a dedup step in middleware. Waiting runs are persisted and survive deploys; a run paused for approval keeps its key and resumes before waiting runs; `queue` starts waiting runs in arrival order with no length cap or wait timeout.

## Approvals

Gate specific tool calls behind a human decision in the dashboard.

```yaml
approval:
  tools:
    - charge_customer
    - admin.delete: context.role == 'admin'    # condition: context.* or param.*
    - refund: param.amount > 50                # gate only when the argument exceeds threshold
  inputs:
    - get_mfa:
        prompt: Use this tool when authentication requires an MFA code.
        label: MFA code
        sensitive: true
        params:
          - reason: str
          - account_email: str
  timeout: 3600           # min 30s, max 604800s
  message: "Approve this action"
  on_rejection: fail      # fail = stop run (default); continue = skip the tool and keep going
```

The run pauses at the gated tool call until a reviewer approves or rejects in **Dashboard → Approvals**. Approval conditions use `param.*` (the tool's arguments) and `context.*`; `input.*` is **not** valid here (unlike tool conditions in `tools:`). If a condition cannot be evaluated because a parameter or context value is missing, connic fails safe and requires approval.

`approval.inputs` is available only on LLM agents. Each entry creates a tool that pauses for required human text and returns that text to the agent; no Python function or top-level `tools` entry is needed. Input-tool names must be unique ASCII identifiers of at most 64 characters, cannot be `search_tools` or `use_tool`, and cannot collide with `tools`, `discoverable_tools`, or `approval.tools`. Generated input tools count toward the 200-tool limit. `params` names must be unique ASCII identifiers, cannot be Python keywords or `context`, and use `str`, `int`, `float`, or `bool`; every declared parameter is required. `sensitive: true` masks the field and protects the response in storage and logs while still returning it to the model.

## Cascading defaults with `_defaults.yaml`

Drop a `_defaults.yaml` into any directory under `agents/` to share configuration with every agent at that directory level and below. Use it for shared models, guardrails, tools, and access rules across families of agents.

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

connic combines `_defaults.yaml` files from `agents/` down to the agent's directory, shallowest first. The agent's own YAML is applied last and wins on conflict.

### What can live in `_defaults.yaml`

The same fields as a normal agent YAML, but partial — only set what you want to share. Two exceptions:

- `name` and `description` are **forbidden** in defaults (they're per-agent identity; sharing them would either be useless or collide).
- `version` is allowed in defaults, but every agent file must also set it along with `name` and `description`.

The linter rejects defaults files that contain `name` or `description`, pointing at the offending file.

### Merge rules

Applied recursively as the chain is folded together:

- **Scalars** (`model`, `temperature`, `system_prompt`, `timeout`, `reasoning_effort`, …) — deeper layer replaces shallower; the agent file replaces all.
- **Dicts** (`access`, `lookups`, `retry_options`, `approval`, `session`, `concurrency`, `guardrails`, `access.database`, `access.files`, …) — recursive deep merge, per-key. Defaults can supply rules for some collections; an agent can add more without losing the inherited ones.
- **Lists** — concat with dedup, so children **add to** rather than replace inherited lists:
  - `tools`, `discoverable_tools`, `approval.tools` — dedup by tool ref. An agent re-declaring an inherited tool (e.g. with a different condition) overrides the inherited entry.
  - `approval.inputs` — dedup by input-tool name. A deeper declaration replaces the inherited configuration for that name.
  - `mcp_servers` — dedup by server `name`. An agent's full server config replaces the inherited one on collision.
  - `guardrails.input`, `guardrails.output` — dedup by rule `name` if present; otherwise appended.
  - Sequential `agents:` — dedup by string.

Order in the final list: inherited entries first (root → deepest), then the agent's own, minus anything the agent re-declared.

### Example

`agents/_defaults.yaml`:

```yaml
version: "1.0"
type: llm
model: connic/gpt-5.6-terra
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

Effective config for `refund-agent`: `model: connic/gpt-5.6-terra`, `temperature: 0`, the two inherited guardrails, and `tools: [audit.log_event, billing.lookup_charge, billing.issue_refund]`. The agent didn't redeclare `audit.log_event`, so it's inherited as-is.

### Tips

- Use `connic lint` to see the final shape if you're unsure what got merged — load errors point at the file whose values caused the rejection.
- Test variants (`<base>-test-<name>.yaml`) inherit the same defaults as their base, since they live in the same directory.

## Common mistakes

- Writing `tools/billing.py` and referencing `billing.lookup_invoice` but the function is named `lookupInvoice` — case and snake_case matter.
- Using `tools: lookup_invoice` (string) instead of `tools: [lookup_invoice]` (list).
- Omitting `version`, `name`, or `description`, or omitting `model` / `system_prompt` from an LLM agent — the linter rejects the file. `description` is required and often forgotten. None of `name`, `description`, or `version` can be supplied by a `_defaults.yaml`; `name` and `description` are forbidden there entirely.
- Setting `session.key` to a literal string instead of a `context.*` / `input.*` path.
- Listing the same tool both unconditionally and conditionally — not allowed.
- Using `input.*` in an `approval.tools` condition — only `param.*` and `context.*` are valid there. Conversely, using `param.*` in a `tools:` conditional — that's only valid in approvals.
