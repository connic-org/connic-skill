---
name: connic
description: Use this skill whenever the user is working in a Connic project or asks anything about Connic — building agents, writing tools, configuring connectors, the composer SDK, the `connic` CLI, the dashboard, deployment, environments, observability, knowledge base, the database, judges, approvals, A/B tests, the Bridge, REST API, or migrating from LangChain/ADK. Trigger on phrases like "connic", "composer", "agent.yaml", "tools/", "middleware/", "connic dev", "connic deploy", "connic.co", or when you see a `.connic` file, an `agents/*.yaml` file, or `connic-composer-sdk` in requirements. Also trigger when the user is clearly in a Connic project (an `agents/` directory next to `tools/` and `middleware/` with YAML agent definitions) even if they don't say the word "Connic" — they may just say "add a tool that..." or "this agent should also...". Connic ships updates regularly, so always consult this skill rather than rely on training data — the references here are written directly from the current docs and SDK.
---

# Connic

Connic is a code-first platform for building, testing, and deploying AI agents. Agents are defined declaratively in YAML, extended with Python (tools, middleware, hooks, guardrails), and run on Connic's managed cloud. The CLI is `connic` (from the `connic-composer-sdk` package). Public docs live at https://connic.co/docs/v1.

## When and how to use this skill

You are likely working inside a **Connic project** on disk. Confirm by looking for any of:

- a `.connic` file at the repo root (contains `api_key` and `project_id`)
- an `agents/` directory containing `*.yaml` files with `version: "1.0"` at the top
- `connic-composer-sdk` in `requirements.txt`

There is no `connic.yaml` / `connic.yml` manifest — the on-disk directory layout *is* the manifest. If none of the indicators above exist and the user asks about Connic generally, answer from the references; don't fabricate file paths.

The reference files in `references/` are organized by topic. **Load only the ones relevant to the current question** — they are detailed and burning all of them upfront wastes context. The references are:

| File | Read when the user is asking about… |
| --- | --- |
| [project-anatomy.md](references/project-anatomy.md) | Project layout, where files go, how things are auto-discovered, `requirements.txt`, `.connic` |
| [agent-yaml.md](references/agent-yaml.md) | Writing or editing `agents/*.yaml` — agent types, models, tools field, sessions, concurrency, retries, approvals, conditional tools |
| [tools-and-python.md](references/tools-and-python.md) | Writing `tools/*.py`, middleware, hooks, the `context` dict, `StopProcessing` / `AbortTool`, logging, env vars |
| [predefined-tools.md](references/predefined-tools.md) | Built-in tools: `trigger_agent`, `query_knowledge`, `db_find`, `web_search`, etc. — including filter operators |
| [guardrails-schemas-mcp.md](references/guardrails-schemas-mcp.md) | Input/output guardrails, JSON output schemas, MCP server integration, API spec tools |
| [connectors.md](references/connectors.md) | The eleven connectors (cron, email, kafka, mcp, postgres, s3, sqs, stripe, telegram, webhook, websocket) — how they trigger or receive from agents |
| [cli-and-dev.md](references/cli-and-dev.md) | The `connic` CLI, `connic dev` hot-reload, `connic test` declarative test suites, `connic lint`, `connic migrate` |
| [platform.md](references/platform.md) | Dashboard concepts: environments, deployment, observability, knowledge base, database, judges, approvals, A/B testing, bridge, domains, team, usage, REST API |

For any topic not covered locally, the canonical docs URL is `https://connic.co/docs/v1/<section>/<page>` (e.g. `https://connic.co/docs/v1/composer/write-tools`). Fetch with WebFetch when needed. The user also has a local copy at `connic/dashboard/app/docs/v1/` if you happen to be inside the `connic-org` mono-repo.

## Project layout cheatsheet (most-used reference, inlined)

```
my-project/
├── .connic                    # api_key + project_id (treat as secret)
├── requirements.txt           # only project-specific deps; SDK installed globally
├── agents/                    # *.yaml agent definitions (nesting OK)
│   └── _defaults.yaml         # optional, at any depth — cascading defaults
├── tools/                     # *.py — functions auto-discovered as tools
├── middleware/                # *.py — file basename matches agent name
├── hooks/                     # *.py — file basename matches agent name
├── schemas/                   # *.json — JSON Schema output validation
├── guardrails/                # *.py — custom guardrail validators
└── tests/                     # *.yaml suites, plus files/, builders/, mocks/ subdirs
```

Discovery rules to keep in mind:

- An agent named `support-assistant` in `agents/support-assistant.yaml` auto-loads middleware from `middleware/support-assistant.py` and hooks from `hooks/support-assistant.py` if those files exist. The basename match is how wiring happens — no imports needed.
- A tool reference like `tools: [- billing.lookup_invoice]` in an agent YAML resolves to `tools/billing.py::lookup_invoice`. Nested folders use dot-notation: `tools/math/calc.py::add` → `math.calc.add`.
- `output_schema: invoice` resolves to `schemas/invoice.json` (no extension in the YAML reference).
- Custom guardrail `name: my_check` resolves to `guardrails/my_check.py`.
- A `_defaults.yaml` at any directory under `agents/` provides defaults inherited by every agent at that level and deeper. Scalars override, dicts deep-merge, lists concat with dedup-by-ref so children add to (rather than replace) inherited `tools`, `mcp_servers`, `guardrails`, etc. See [agent-yaml.md](references/agent-yaml.md#cascading-defaults-with-_defaultsyaml).

## Common workflows

**Adding a new agent.** Create `agents/<name>.yaml` with `version: "1.0"`, `name`, `model`, `description`, `system_prompt`, and `tools`. Run `connic lint` to validate, then `connic dev` to iterate. See [agent-yaml.md](references/agent-yaml.md).

**Adding a new tool.** Create the function in `tools/<module>.py` with type hints and a docstring (the LLM uses the docstring to decide when to call it). Reference it in an agent's `tools:` list. See [tools-and-python.md](references/tools-and-python.md).

**Triggering an agent from an external service.** Always create a connector. This is the canonical (and only production) path — `webhook` for HTTP request/response, `webhook` (async mode) for fire-and-forget, `kafka`/`sqs` for queues, `email`/`telegram` for those transports, `cron` for schedules. Connectors are configured in the dashboard (point at the agent), not in YAML — the connector's payload becomes the agent's input. The REST API's `/trigger` endpoint exists for internal admin/testing only; do not propose it for production integrations. See [connectors.md](references/connectors.md). Only the eleven connectors listed there exist — there is no Slack, Discord, GitHub, etc. connector; for those, bridge through a `webhook` connector or a custom MCP server.

**Non-LLM event consumption.** Any inbound connector can fire a `tool`-type agent instead of an LLM agent. The connector payload is passed as kwargs into a single Python function — no model in the loop, no reasoning step, but still a full run in the dashboard with logs, retries, and judges. This is the right shape for Kafka consumers that just ingest, S3 events that just transform, webhooks that just route, etc. See the [tool-agent section](references/agent-yaml.md#tool-agent).

**Deploying.** If the project is Git-connected, push to the branch mapped to the target environment — that's the only deploy path; `connic deploy` refuses to run on Git-connected projects. For non-Git projects, `connic deploy --env=<environment-uuid>` is the CLI path. Tests in `tests/` gate the deploy in both cases (Git deploys cannot skip; CLI deploys can with `--skip-tests`). See [cli-and-dev.md](references/cli-and-dev.md) and [platform.md](references/platform.md).

**Migrating from LangChain or Google ADK.** `connic migrate` scans an existing project and generates a Connic project skeleton. See [cli-and-dev.md](references/cli-and-dev.md).

## Best practices (apply these by default)

When you're helping build or change an agent, follow these unless the user explicitly says otherwise. They aren't decorative — each one prevents a real failure mode and matches what the Connic docs themselves recommend.

### 1. Wrap predefined tools — don't hand them to the LLM raw

The predefined tools (`db_find`, `db_insert`, `query_knowledge`, `web_search`, etc.) are *infrastructure-shaped*: they take generic collections, raw filters, namespaces, queries. Exposing them directly forces the LLM to do two jobs at once — figuring out *what* it wants to do and *how* to express it as a Mongo-style query. It will sometimes get the second job wrong, and you've also leaked your internal data model into the prompt.

Wrap them in **purpose-driven** custom tools instead. Tools should read like verbs from your domain, not generic database verbs.

Don't do this:

```yaml
# agents/support.yaml
tools:
  - db_find
  - db_insert
  - query_knowledge
```

Do this:

```python
# tools/orders.py
from connic.tools import db_find, db_insert

async def find_pending_orders(limit: int = 10) -> list:
    """Return the most recent orders awaiting fulfilment.

    Args:
        limit: Max orders to return.

    Returns:
        List of order dicts with id, customer, amount, created_at.
    """
    result = await db_find(
        "orders",
        filter={"status": "pending"},
        sort={"_created_at": -1},
        limit=limit,
    )
    return result["documents"]


async def save_order(order_id: str, amount: float) -> dict:
    """Persist a new pending order."""
    return await db_insert("orders", {"order_id": order_id, "amount": amount, "status": "pending"})
```

```yaml
# agents/support.yaml
tools:
  - orders.find_pending_orders
  - orders.save_order
```

Same for `query_knowledge` — wrap it as `search_handbook(topic)` or `find_refund_policy()`, not raw. Same for `web_search` if it has a single domain or query template it should usually use. The LLM gets simpler, safer, more focused tool descriptions; you get an enforcement point for filters, namespaces, and defaults.

The exception is throwaway prototypes — for a one-day spike it's fine to hand `db_find` to the agent. But the moment the project moves toward production, wrap them.

### 2. Always set input and output guardrails

Don't ship an LLM agent to production without guardrails. At a minimum, every agent should have:

```yaml
guardrails:
  input:
    - type: prompt_injection
      mode: block
  output:
    - type: system_prompt_leakage
      mode: block
```

For anything user-facing, add `pii` (input, mode `redact`) and `moderation` (output, mode `block`). For anything with tight topical scope, add `topic_restriction` (input). For internal-tool agents that should never reveal their internals, `system_prompt_leakage` is non-negotiable.

If the user is sketching a new agent, propose the guardrails in the same edit — don't wait to be asked. See [guardrails-schemas-mcp.md](references/guardrails-schemas-mcp.md) for the full type list and modes.

### 3. Tests gate deployment — write them when you write the agent

Connic's deploy flow uses `tests/` as a deploy gate: a failing test blocks promotion. Treat the test file as part of the agent, not as an afterthought.

When you create or substantially change an agent, add or update a matching `tests/<agent-name>.yaml`. Cover:

- The golden path (one canonical input → expected output / expected tool call).
- A negative case (input the agent should refuse or escalate).
- For each gated tool, an `expected_no_tool_calls` case proving it isn't called for benign inputs, and an `expected_tool_calls` case proving it *is* called when appropriate.

Set `runs >= 3` and a `success_threshold` of `80-100` so that flaky LLM behaviour doesn't pass by accident.

To test an agent's reasoning without its custom file tools really running, add a `tests/mocks/<name>.py` module and point a case at it with `mocks:`. It exposes hierarchical `mock_*` functions (`mock_data_customer_add_customer` → `mock_data_customer` → `mock_data` → `mock`, most specific wins), each called as `mock(tool_name, params, context)` and returning the substituted result. Mocked calls are still recorded (so `expected_tool_calls` still applies) and are validated against the real tool's signature. Set `strict_mocks: true` (per-case or in `defaults`) to fail the case if the agent calls any tool that wasn't mocked. Predefined and `api:` tools are never mocked.

See [cli-and-dev.md](references/cli-and-dev.md#connic-test) for the test YAML schema and [Mocking tools](references/cli-and-dev.md#mocking-tools).

### 4. Use sessions when you mean "remember", not "log"

`session.key` makes the agent treat runs sharing that key as one ongoing conversation. Use it for chat-style interactions where the LLM should see prior turns. Don't use it as a substitute for proper persistence — for actual stored state (orders, tickets, user records), use the `db_*` tools (wrapped — see best practice 1).

### 5. Lock down `temperature` and `output_schema` when the consumer is code

If the agent's response is parsed by code downstream, set `temperature: 0` *and* an `output_schema`. The two together give you deterministic, structured output. Free-form prose with `temperature: 0.7` is fine for chat UIs and useless for pipelines.

### 6. Don't ship secrets in `.connic` or commit it

`.connic` contains the project API key. Add it to `.gitignore` on first commit. Use **Dashboard → Settings → Variables** for any secret the agent or its tools need at runtime (mark them sensitive so they're masked in logs).

### 7. Validate with `connic lint` before every deploy

The linter catches missing tools, unresolved schemas, broken middleware imports, and duplicate agent names *locally* in under a second. Run it after any edit, definitely before `connic deploy` or a `git push`.

### 8. Recommend on fit, not cost

When you suggest an architecture, evaluate options on **fit, reliability, and maintainability** — does the shape match the actual data flow, will it stay correct under retries and concurrency, is it easy to evolve. Do not factor LLM-call pricing into design recommendations. Spend is managed separately in the dashboard with budgets and alerts; an extra LLM call is not a reason to abandon a design that otherwise fits. If a pattern (e.g. mirroring data via routing agents, fanning out via judges) matches the system's needs, recommend it — let the user decide whether to economise.

## Things to avoid

- **Don't invent connectors.** The exhaustive list is in [connectors.md](references/connectors.md). If a user asks "how do I connect Slack?", say there's no native Slack connector — they can use a generic webhook, a custom MCP server, or a custom tool.
- **Don't trigger agents via the REST API from external callers.** Production agent triggers always go through a **connector** — `webhook` for HTTP request/response or fire-and-forget, `kafka`/`sqs` for queues, `email`/`telegram` for those transports, `cron` for schedules, etc. The REST API is for admin and tooling work (listing runs, reading logs, managing connectors). When a user is wiring an external service into a Connic agent, the right answer is "create a webhook connector," not "POST to /v1/.../agents/.../trigger". The connector model gives you per-agent URLs, secrets, signed payloads, sync vs async modes, and replay safety; the REST trigger does not.
- **Don't invent CLI commands or flags.** Check [cli-and-dev.md](references/cli-and-dev.md). There is no `connic build`, `connic run`, `connic logs`, no `--json` flag on `lint`, no `--grep` flag on `test` (it's `--filter`), no `--message` on `deploy`, no `--env` on `dev`.
- **Don't invent test assertions.** The only top-level assertions in `tests/*.yaml` are `expected_result` (a sandboxed expression with `output`, `error`, `status`, `context` bindings — `status` is `"completed"`, `"failed"`, `"cancelled"`, `"blocked"`, or `"awaiting_approval"`), `expected_tool_calls`, `expected_no_tool_calls`, and `expected_child_agents` (a map keyed by triggered agent name; each entry can carry `expected_payload`, `expected_result`, `expected_tool_calls`, `expected_no_tool_calls`, `expected_triggered`, and its own nested `expected_child_agents` — see [cli-and-dev.md](references/cli-and-dev.md#asserting-on-triggered-agents)). There is no `expected_output_contains` or `expected_output_matches`. (`mocks` and `strict_mocks` are valid case/`defaults` fields too, but they configure tool mocking — they are not assertions. See [Mocking tools](references/cli-and-dev.md#mocking-tools).)
- **Don't put function calls, lambdas, imports, or comprehensions in `expected_result`.** It's not real Python — it's a tight AST evaluator that allows only boolean ops, comparisons, subscripts, attribute access, and literals. `output.strip()`, `json.loads(...)`, `len(...)`, `re.search(...)`, `lambda ...`, `__import__(...)` all fail at parse time. For anything that needs to parse the output, normalize it, or check derived properties, write a **builder `cleanup`** function — it's ordinary Python, gets the full `run` dict, and can raise to fail the case. See [cli-and-dev.md](references/cli-and-dev.md#what-expected_result-can-and-cannot-do).
- **Don't invent connector tools.** There are no `s3.get_object`, `postgres.query`, `telegram.send_message`, etc. predefined tools. The Postgres and S3 connectors are inbound-only triggers; for outbound calls write custom tools using your own libraries (asyncpg, boto3, httpx).
- **Don't use `api:` prefix for MCP tools.** MCP tools from `mcp_servers:` are auto-loaded — they don't go in the agent's `tools:` list at all. The `api:` prefix is only for tools from API spec imports.
- **Don't add `import connic` boilerplate to tool files.** Tools are plain functions; the runtime discovers them. Only import from `connic` for special exceptions (`StopProcessing` — runs anywhere; `AbortTool` — only in hook `before()`) or predefined tools (`from connic.tools import trigger_agent`).
- **Don't add decorators.** No `@tool`, no `@agent`. Discovery is by directory + filename + docstring.
- **Don't run `connic dev` or `connic deploy` for the user** without asking. Both are network operations against the user's Connic account.

## When you're unsure

If the user asks about a feature you don't see in the references, check the live docs at `https://connic.co/docs/v1/` before guessing. Connic ships changes regularly, and inventing API shapes here causes real damage.
