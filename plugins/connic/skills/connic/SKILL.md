---
name: connic
description: Use when the user works in a Connic project or asks about Connic agents, Connic MCP, `mcp.connic.co`, live project inspection or operations, `connic/*` or BYOK models, tools, connectors, Composer SDK, the `connic` CLI, Project credit and billing, deployment, environments, observability, Retrieval, databases, judges, approvals, A/B tests, AI Governance, the Bridge, REST API, or LangChain/ADK migration. Trigger on "connic", "composer", "agent.yaml", "tools/", "middleware/", "connic dev", "connic deploy", "connic.co", `.connic`, `agents/*.yaml`, or `connic-composer-sdk`. Also trigger in a Connic project — identified by an `agents/` directory beside `tools/` and `middleware/` — even when the user only asks to add a tool or change an agent.
metadata:
  version: "1.2.4"
---

# Connic

Connic is a code-first platform for building, testing, and deploying AI agents. Agents are defined declaratively in YAML, extended with Python (tools, middleware, hooks, guardrails), and run on Connic's managed cloud. The CLI is `connic` (from the `connic-composer-sdk` package). Public docs live at https://connic.co/docs/v1.

## Start-of-session update check

For local Connic project, CLI, or SDK work, run `connic update --check` once per agent session when the CLI is available. Skip this for an MCP-only session, standalone platform guidance, or when the CLI is not installed. If it reports updates, tell the user and offer `connic update` for both SDK and skill, `connic update --sdk` for the SDK only, or `connic update --skill` for the skill only. Do not install an update without the user's explicit consent.

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
| [tools-and-python.md](references/tools-and-python.md) | Writing `tools/*.py`, returning files with `ToolFile`, middleware, hooks, the `context` dict, `StopProcessing` / `AbortTool`, logging, env vars |
| [predefined-tools.md](references/predefined-tools.md) | Built-in tools: `trigger_agent`, `retrieval_query`, `db_find`, `web_search`, etc. — including filter operators |
| [guardrails-schemas-mcp.md](references/guardrails-schemas-mcp.md) | Input/output guardrails, JSON output schemas, agents consuming external MCP servers, API spec tools |
| [connectors.md](references/connectors.md) | Built-in connectors (cron, email, kafka, mcp, postgres, s3, sqs, stripe, telegram, webhook, websocket) — how they trigger or receive from agents |
| [cli-and-dev.md](references/cli-and-dev.md) | The `connic` CLI, `connic dev` hot-reload, `connic test` declarative test suites, `connic lint`, `connic migrate` |
| [ab-testing.md](references/ab-testing.md) | A/B test variants, Confidence and Exploratory modes, traffic assignment, safety rules, results, and lifecycle |
| [ai-governance.md](references/ai-governance.md) | AI systems, assessments, controls, Article 50 records, incidents, evidence snapshots, and governance API |
| [platform.md](references/platform.md) | Dashboard concepts: models, billing, environments, deployment, observability, Retrieval, database, judges, approvals, bridge, domains, team, usage, REST API |
| [platform-mcp.md](references/platform-mcp.md) | Connecting an AI client to Connic MCP, live project/environment operations, OAuth scopes, tool selection, and revocation |

For any topic not covered locally, the canonical docs URL is `https://connic.co/docs/v1/<section>/<page>` (e.g. `https://connic.co/docs/v1/build/tools`). Fetch with WebFetch when needed.

### Choosing live tools or standalone guidance

The ChatGPT/Codex, Claude, and Cursor plugin packages bundle the production MCP endpoint; standalone skill installations do not modify the client's MCP configuration. When Connic MCP tools are available, use them for authenticated live platform state and operations within the granted scope. Discover the available tools from the live authorization and use only the narrowest relevant one. Perform writes only when explicitly requested. Otherwise remain fully useful with the references, project files, CLI, dashboard, and REST API. Do not claim to have inspected live state when no Connic MCP tool is available.

Connic MCP is the platform-management server. It is different from an agent's `mcp_servers:` configuration, where the agent consumes an external server, and from the MCP connector, where an external client invokes a deployed Connic agent. See [platform-mcp.md](references/platform-mcp.md).

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

**Adding a new agent.** Create `agents/<name>.yaml` with `version: "1.0"`, `name`, and `description`, then add the fields its type requires: `model` and `system_prompt` for an LLM agent, `agents` for a sequential agent, or `tool_name` for a tool agent. `tools` is optional and only applies to LLM agents. Run `connic lint` to validate, then use the user's existing development workflow to iterate. See [agent-yaml.md](references/agent-yaml.md).

**Adding a new tool.** Create the function in `tools/<module>.py` with type hints and a docstring (the LLM uses the docstring to decide when to call it). Reference it in an agent's `tools:` list. See [tools-and-python.md](references/tools-and-python.md).

**Triggering an agent from an external service.** Use a connector — `webhook` for HTTP request/response or fire-and-forget, `kafka`/`sqs` for queues, `email`/`telegram` for those transports, and `cron` for schedules. Connectors provide transport-specific endpoints, authentication, sync/async behavior, and delivery semantics; do not assume generic deduplication or replay protection. The REST API is for project management, not event-driven agent runs. See [connectors.md](references/connectors.md). Only the connectors listed there exist — there is no native Slack, Discord, or GitHub connector; bridge those through a webhook, MCP server, or custom tool.

**Non-LLM event consumption.** Any inbound connector can fire a `tool`-type agent instead of an LLM agent. Connic passes one normalized dict to the tool's required `payload` parameter, plus `context` when declared; it never expands payload keys into separate arguments. There is no model or reasoning step, but the run still has logs, retries, and judges. This fits Kafka consumers that ingest, S3 events that transform, and webhooks that route. See the [tool-agent section](references/agent-yaml.md#tool-agent).

**Deploying.** If the project is Git-connected, push to the branch mapped to the target environment — that's the only deploy path; `connic deploy` refuses to run on Git-connected projects. For non-Git projects, bare `connic deploy` targets the default environment and `--env=<environment-uuid>` overrides it. Tests in `tests/` gate the deploy in both cases (Git deploys cannot skip; CLI deploys can with `--skip-tests`). See [cli-and-dev.md](references/cli-and-dev.md) and [platform.md](references/platform.md).

**Migrating from LangChain or Google ADK.** `connic migrate` scans an existing project and generates a Connic project skeleton. See [cli-and-dev.md](references/cli-and-dev.md).

## Best practices (apply these by default)

When you're helping build or change an agent, follow these unless the user explicitly says otherwise.

### 1. Wrap predefined tools — don't hand them to the LLM raw

The predefined tools (`db_find`, `db_insert`, `retrieval_query`, `web_search`, etc.) take generic collections, filters, namespaces, and queries. Exposing them directly makes the LLM choose both the user action and the storage-level request. A purpose-driven wrapper keeps project data structures out of the prompt and constrains the operation.

Wrap them in **purpose-driven** custom tools instead. Tools should read like verbs from your domain, not generic database verbs.

Don't do this:

```yaml
# agents/support.yaml
tools:
  - db_find
  - db_insert
  - retrieval_query
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

Same for `retrieval_query` — wrap it as `search_handbook(topic)` or `find_refund_policy()`, not raw. Same for `web_search` if it has a single domain or query template it should usually use. The LLM gets simpler, safer, more focused tool descriptions; you get an enforcement point for filters, namespaces, and defaults.

The exception is throwaway prototypes — for a one-day spike it's fine to hand `db_find` to the agent. But the moment the project moves toward production, wrap them.

### 2. Always set input and output guardrails

Don't ship an LLM agent to production without guardrails. At a minimum, every agent should have:

```yaml
guardrails:
  input:
    - type: prompt_injection
      mode: block
    - type: pii
      mode: redact
  output:
    - type: moderation
      mode: block
    - type: system_prompt_leakage
      mode: block
```

This is the documented production baseline: prompt-injection detection plus PII redaction on input, and moderation plus system-prompt-leakage protection on output. For anything with tight topical scope, add `topic_restriction` (input).

If the user is sketching a new LLM agent, propose the guardrails in the same edit — don't wait to be asked. Tool agents have no LLM boundary; validate their payload and side effects in Python instead. See [guardrails-schemas-mcp.md](references/guardrails-schemas-mcp.md) for the full type list and modes.

### 3. Tests gate deployment — write them when you write the agent

Connic's deploy flow uses `tests/` as a deploy gate: a failing test blocks promotion. Treat the test file as part of the agent, not as an afterthought.

When you create or substantially change an agent, add or update a matching `tests/<agent-name>.yaml`. Cover:

- The golden path (one canonical input → expected output / expected tool call).
- A negative case (input the agent should refuse or escalate).
- For each gated tool, an `expected_no_tool_calls` case proving it isn't called for benign inputs, and an `expected_tool_calls` case proving it *is* called when appropriate.

Keep the deterministic default of `runs: 1` and `success_threshold: 100`. For genuinely stochastic behavior, raise `runs` (commonly 3–5) and lower the threshold only as far as the product's acceptable pass rate.

To test an agent's reasoning without selected custom code really running, add a `tests/mocks/<name>.py` module and point a case at it with `mocks:`. Tool results use hierarchical `mock_*` functions (`mock_data_customer_add_customer` → `mock_data_customer` → `mock_data` → `mock`, most specific wins), each called as `mock(tool_name, params, context)`. Predefined and `api:` tool implementations always run for real.

The same module can replace `middleware_before` / `middleware_after`, hierarchical tool-hook phases ending in `_hook_before` / `_hook_after`, and custom guardrails (`guardrail_input_<name>` → `guardrail_input` → `guardrail`, with the equivalent output ladder). Lifecycle replacements mirror the real function signatures. A match replaces an existing phase; it does not add a missing one. Without a match, the real code runs by default. Built-in guardrails are never mocked. `strict_mocks: true` remains tool-only. Enable `strict_hook_mocks`, `strict_middleware_mocks`, or `strict_guardrail_mocks` independently in `defaults` or per case to fail before an unmatched configured eligible real phase executes; all default to `false`, and missing phases and built-in guardrails are exempt.

To test HITL end to end, add `approval_decisions` to the case. Each entry has a canonical `tool`, `decision: approve | reject | timeout`, an optional `reason`, and an optional safe `params` expression whose bindings are `params` and the builder `context`. Connic applies the scripted decision and resumes the same run when the approval configuration permits. Entries are consumed at most once per invocation. With `strict_approval_decisions: false` (the default), an unmatched pending approval returns `status == "awaiting_approval"`, and unused entries are ignored. Set it to `true` per case or in `defaults` to fail on either condition.

See [cli-and-dev.md](references/cli-and-dev.md#connic-test) for the test YAML schema and [Mocking tools](references/cli-and-dev.md#mocking-tools).

### 4. Use sessions when you mean "remember", not "log"

`session.key` makes the agent treat runs sharing that key as one ongoing conversation. Use it for chat-style interactions where the LLM should see prior turns. Don't use it as a substitute for proper persistence — for actual stored state (orders, tickets, user records), use the `db_*` tools (wrapped — see best practice 1).

### 5. Lock down `temperature` and `output_schema` when the consumer is code

If the agent's response is parsed by code downstream, set `temperature: 0` *and* an `output_schema`. The two together make output more repeatable and structurally validated; they do not make model behavior mathematically deterministic. Free-form prose with a higher temperature is appropriate for chat UIs, not brittle machine-parsed pipelines.

### 6. Don't ship secrets in `.connic` or commit it

`.connic` contains the project API key. Add it to `.gitignore` on first commit. Use **Dashboard → Settings → Variables** for any secret the agent or its tools need at runtime (mark them sensitive so they're masked in logs).

Never read, print, copy, or pass `.connic` credentials as MCP tool arguments. A Connic MCP client authenticates through its own OAuth flow; the skill supplies workflow knowledge and never handles its access or refresh tokens.

### 7. Validate with `connic lint` before every deploy

The linter catches missing tools, unresolved schemas, broken middleware imports, and duplicate agent names locally. Run it after any edit and before `connic deploy` or a `git push`.

### 8. Recommend on fit first

When you suggest an architecture, evaluate **fit, reliability, and maintainability** first — does the shape match the actual data flow, will it stay correct under retries and concurrency, is it easy to evolve. Then compare latency and Project-credit or provider cost among options that meet those requirements. For example, model-backed guardrails such as topic restriction and relevance can use a smaller suitable model. Do not sacrifice correctness merely to remove an LLM call; surface the tradeoff so the user can decide.

## Things to avoid

- **Don't invent connectors.** The exhaustive list is in [connectors.md](references/connectors.md). If a user asks "how do I connect Slack?", say there's no native Slack connector — they can use a generic webhook, a custom MCP server, or a custom tool.
- **Don't invent model IDs.** Select managed models from the live [Connic Model Catalog](https://connic.co/docs/v1/build/connic-models). `connic/*` models need no provider key; BYOK model IDs must be supported by the configured provider.
- **Don't wire event-driven agent runs through the REST API.** Use the connector matching the transport.
- **Don't invent CLI commands or flags.** Check [cli-and-dev.md](references/cli-and-dev.md). There is no `connic build`, `connic run`, `connic logs`, no `--json` flag on `lint`, no `--grep` flag on `test` (it's `--filter`), no `--message` on `deploy`, no `--env` on `dev`.
- **Don't invent test assertions.** The only top-level assertions in `tests/*.yaml` are `expected_result` (a sandboxed expression with `output`, `error`, `status`, `context` bindings — `status` is `"completed"`, `"failed"`, `"cancelled"`, `"blocked"`, or `"awaiting_approval"`), `expected_tool_calls`, `expected_tool_call_order`, `expected_no_tool_calls`, and `expected_child_agents` (a map keyed by triggered agent name; each entry can carry `expected_payload`, `expected_result`, `expected_tool_calls`, `expected_tool_call_order`, `expected_no_tool_calls`, `expected_triggered`, and its own nested `expected_child_agents` — see [cli-and-dev.md](references/cli-and-dev.md#asserting-on-triggered-agents)). There is no `expected_output_contains` or `expected_output_matches`. (`mocks`, the four independent `strict_*_mocks` flags, `approval_decisions`, and `strict_approval_decisions` are valid execution controls, not assertions. See [Mocking tools](references/cli-and-dev.md#mocking-tools) and [Testing approvals](references/cli-and-dev.md#testing-approvals-hitl).)
- **Don't put function calls, lambdas, imports, or comprehensions in `expected_result`.** It's not real Python — it's a tight AST evaluator that allows only boolean ops, comparisons, subscripts, attribute access, and literals. `output.strip()`, `json.loads(...)`, `len(...)`, `re.search(...)`, `lambda ...`, `__import__(...)` all fail at parse time. For anything that needs to parse the output, normalize it, or check derived properties, write a **builder `cleanup`** function — it's ordinary Python, gets the full `run` dict, and can raise to fail the case. See [cli-and-dev.md](references/cli-and-dev.md#what-expected_result-can-and-cannot-do).
- **Don't invent connector tools.** There are no `s3.get_object`, `postgres.query`, `telegram.send_message`, etc. predefined tools. The Postgres and S3 connectors are inbound-only triggers; for outbound calls write custom tools using your own libraries (asyncpg, boto3, httpx).
- **Don't use `api:` prefix for MCP tools.** MCP tools from `mcp_servers:` are auto-loaded — they don't go in the agent's `tools:` list at all. The `api:` prefix is only for tools from API spec imports.
- **Don't confuse Connic MCP with agent MCP integrations.** Connic MCP lets an AI client manage a Connic project. `mcp_servers:` lets a Connic agent consume external tools. The MCP connector lets an external client invoke deployed agents.
- **Don't add `import connic` boilerplate to tool files.** Tools are plain functions that Connic discovers automatically. Import from `connic` only for SDK types such as `ToolFile` and special exceptions (`StopProcessing` — runs anywhere; `AbortTool` — only in hook `before()`), or import predefined tools from `connic.tools` (for example, `trigger_agent`).
- **Don't add decorators.** No `@tool`, no `@agent`. Discovery is by directory + filename + docstring.
- **Don't run `connic dev` or `connic deploy` for the user** without asking. Both are network operations against the user's Connic account.

## When you're unsure

If the user asks about a feature you don't see in the references, check the docs at `https://connic.co/docs/v1/` before answering. Do not invent API shapes.
