# The `connic` CLI and dev workflow

The CLI ships with `connic-composer-sdk`. Install with `pip install connic-composer-sdk` (requires Python 3.10+).

## Full command list

| Command | What it does |
| --- | --- |
| `connic init [name]` | Scaffold a new project directory. `--templates=invoice,customer-support` seeds starter templates; `--skill` installs this skill and offers full plugins for detected Codex and Claude Code clients. |
| `connic skill` | Install the project skill under `.agents/skills/connic/` and `.claude/skills/connic/`, then offer full plugins for detected Codex and Claude Code clients. |
| `connic update [--check|--sdk|--skill|--enable-reminders]` | Check for or install available SDK and skill updates, or enable automatic reminders. |
| `connic login` | Browser-based auth; writes `.connic` (api_key + project_id) into the current directory. `--token <project_id>:<api_key>` skips the browser for CI. |
| `connic lint` | Validate YAML, tool references, schemas, middleware/hooks discovery — locally, no upload. |
| `connic tools` | List the custom Python tools discovered in the current project. |
| `connic dev [name]` | Open a cloud dev environment, sync local files, hot-reload on save. Named sessions persist; unnamed are ephemeral. |
| `connic test` | Run declarative test suites from `tests/` against an environment. `--env <id>` picks the environment; `--filter <substring>` runs a subset; `--coverage` runs a static no-network coverage report; `--json` emits machine-readable output. |
| `connic deploy` | Deploy current files to a Connic environment. Refuses to run if the project is connected to a Git repo (use `git push` in that case). |
| `connic migrate [--source <path>] [--dest <path>]` | Scan a LangChain or Google ADK project and emit a Connic-shaped project skeleton; omitted paths are prompted for. |

Run any with `--help` for the canonical flag list.

## `connic init`

```bash
connic init my-agents --skill
# Or seed templates too:
# connic init my-agents --templates=invoice,customer-support --skill
cd my-agents
```

The default scaffold creates `agents/`, `tools/`, `middleware/`, `schemas/`, plus `.gitignore`, `requirements.txt`, and a `README.md` containing a starter example. It does **not** create a `.connic` file (that's `connic login`'s job) and it does **not** drop a stub agent YAML on disk — the README shows what one should look like. `--skill` adds `.agents/skills/connic/` and `.claude/skills/connic/`; omit it when the project should not carry the skill. In an interactive terminal, the command then detects Codex and Claude Code and asks separately whether to install the full Connic plugin. Declining keeps the skill-only setup.

## `connic skill`

```bash
cd my-project
connic skill
```

Installs the Connic skill in `.agents/skills/connic/` and `.claude/skills/connic/`. It is the existing-project equivalent of `connic init --skill`. In an interactive terminal, it also offers to install the full plugin for each detected Codex or Claude Code client. The plugin bundles the skill and Connic MCP; the local copies remain skill-only. See [AI agent setup](https://connic.co/docs/v1/ai-agent-setup).

## `connic update`

```bash
connic update --check
connic update
connic update --sdk
connic update --skill
connic update --enable-reminders
```

Use `--check` to report updates without installing them. With no component flag, `connic update` updates every available component; `--sdk` and `--skill` limit it to one component. Use `--enable-reminders` to set automatic update reminders to enabled.

## `connic login`

```bash
cd my-project
connic login                                          # interactive (opens browser)
connic login --token <project_id>:<api_key>           # CI-friendly, no browser
```

Either form writes `.connic` to the current directory with the project id and API key. Add `.connic` to `.gitignore`.

## `connic lint`

```bash
connic lint
```

Catches:

- Invalid YAML.
- Missing required agent fields (`version`, `name`, `description`, plus `model` and `system_prompt` for LLM agents).
- Tool references that don't resolve (`tools: [billing.missing_function]`).
- Duplicate agent names across files.
- Schema files referenced by `output_schema:` that don't exist or aren't valid JSON.
- Middleware/hooks/guardrails modules that fail to import.

Run this before every deploy. There is no `--json` flag — `lint` only takes `--verbose` / `-v`.

## `connic tools`

```bash
connic tools
```

Prints every custom function discovered under `tools/`, grouped by module. Use this to confirm:

- A new function in `tools/` is discoverable.
- A wildcard like `billing.*` resolves to the functions you expected.
- A function isn't accidentally exposed (e.g. because it doesn't start with `_`).

## `connic dev`

The main iteration loop.

```bash
connic dev                    # ephemeral session, auto-cleaned on exit
connic dev my-feature         # named session, persists between runs
```

Behavior:

- Opens an isolated cloud development environment for the project.
- Syncs `agents/`, `tools/`, `middleware/`, `hooks/`, `schemas/`, `guardrails/`, `tests/`, and `requirements.txt`. Changes to `requirements.txt` trigger a re-install on the next sync — no restart needed. `tests/` is synced too, so you can press `t` in the dev session to run the suites against the active environment.
- Watches local files and syncs edits automatically.
- The dev environment has its own variables, database, Retrieval data, and connectors, separated from standard environments. Unnamed environments are deleted on exit; named environments and their data persist so you can reattach later.
- In an interactive terminal, `r` uploads immediately, `t` runs `tests/` against the active dev environment, and `q` stops with normal cleanup (`Ctrl+C` is the fallback).
- Only one process can attach to a given named dev environment at a time. Use another name or an unnamed session for parallel work.

The `.connic` file is **not** synced — it's local auth only.

For CI or shared shells, authentication can instead come from `CONNIC_API_KEY` and `CONNIC_PROJECT_ID` environment variables.

`connic dev` does not accept an `--env` flag. Use `connic test --env <environment-id>` to run suites against a standard environment.

## `connic test`

Run all suites in `tests/` against the configured environment.

YAML suites are discovered recursively under `tests/`. The filename stem targets the agent (`tests/support.yaml` → `support`); a top-level `agent:` overrides that default so one agent can have multiple suite files. Suite `version` defaults to `"1.0"`. The execution defaults are `runs: 1` (range 1–100), `success_threshold: 100` (1–100), and `timeout_s: 120` (1–3600 seconds). Every case needs `payload` unless it sets `builder`.

```yaml
# tests/sentiment.yaml
version: "1.0"

defaults:
  runs: 5                  # explicit stochastic-test override; default is 1
  success_threshold: 80    # explicit pass-rate override; default is 100
  timeout_s: 60            # 1–3600

tests:
  - name: clearly_positive
    payload: '{"text": "I love this product!"}'
    expected_result: 'status == "completed" and "positive" in output'

  - name: uses_calculator
    payload: '{"text": "what is 2+2"}'
    expected_result: status == "completed"
    expected_tool_calls:
      - math.calculator.add: invocations >= 1

  - name: never_calls_admin_tool
    payload: '{"text": "delete everything"}'
    expected_result: status == "completed"
    expected_no_tool_calls:
      - admin.delete

  - name: looks_up_then_notifies
    payload: '{"order_id": "ORD-1"}'
    expected_tool_call_order:
      - orders.lookup
      - notifications.send

  - name: extract_invoice
    files:
      - invoice.pdf        # resolved from tests/files/invoice.pdf
    expected_result: '"Invoice #" in output'
```

Available assertion fields (these are the only ones):

- `expected_result` — an expression evaluated against the bindings `output`, `error`, `status`, `context` (plus `true` / `false` / `null`). See "What `expected_result` can and cannot do" below.
- `expected_tool_calls` — bare tool names or `tool_name: <expr>` mappings. Names match either a local function name or its qualified ref. Expressions can use `invocations`, `params`, and builder `context`; repeat a tool in separate entries to require distinct argument matches.
- `expected_tool_call_order` — tool names that must appear in this relative order in the trace; unrelated calls may occur between them.
- `expected_no_tool_calls` — list of tool names that must not be called.
- `expected_child_agents` — map of triggered agent name → assertions for that child run (`expected_payload`, `expected_result`, `expected_tool_calls`, `expected_tool_call_order`, `expected_no_tool_calls`, `expected_triggered`, plus a nested `expected_child_agents`). See "Asserting on triggered agents" below.

There are no `expected_output_contains` / `expected_output_matches` fields. Don't invent them.

Fixtures live in `tests/files/`. Reference them with `files: [<bare-filename>, ...]` (plural — never `file:`). Only bare filenames are accepted: path separators and `..` are rejected. The total upload budget is 25 MB, of which code/config outside `tests/files/` remains capped at 5 MB. MIME type is inferred from the extension and falls back to `application/octet-stream`; a missing fixture fails before the agent runs.

If `payload` is a JSON object (or comes from a `builder` that returns a dict), its keys sit at the top level of `context["payload"]`. Otherwise the string is delivered as `{message: <payload>}`. Attached `files` are added alongside, under a `files` list.

Per-case overrides for `runs`, `success_threshold`, and `timeout_s` are allowed.

Case execution controls (not assertions) include `mocks` (a `tests/mocks/<name>.py` module that can replace custom file tools and lifecycle phases), the independent `strict_mocks`, `strict_hook_mocks`, `strict_middleware_mocks`, and `strict_guardrail_mocks` booleans (all also settable in `defaults`), `approval_decisions` (scripted HITL responses), and `strict_approval_decisions` (bool, also settable in `defaults`). See "Testing approvals" and "Mocking tools" below.

### What `expected_result` can and cannot do

`expected_result` is **not** ordinary Python — it's a sandboxed expression evaluator with a very narrow AST surface. Use it for cheap shape checks; fall back to the builder's `cleanup` for anything more.

**Allowed**: boolean ops (`and`, `or`, `not`), comparisons (`==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `not in`, `is`, `is not`), subscripts (`output[...]`, `context["x"]`), attribute access (no underscore attrs — `output.id` works on a dict output and is equivalent to `output["id"]`), list/tuple/set/dict literals, and the names `output`, `error`, `status`, `context`, `true`, `false`, `null`. Expressions are limited to 2,000 characters. Unknown root names, arithmetic, calls, imports, assignments, and private attributes are rejected.

**Binding values:**

- `output` — the agent's output. JSON-parsed when the agent returned valid JSON, otherwise the raw string. So both `output.id == 10` and `"hi" in output` are valid shapes depending on what the agent returns.
- `error` — the run's error string, or `null` when the run did not error.
- `status` — one of `"completed"`, `"failed"`, `"cancelled"`, `"blocked"` (an input/output guardrail intercepted the run), `"awaiting_approval"`.
- `context` — the builder's mutable dict, or `{}` when the case has no builder.

**Short-circuit semantics:** by default an agent error fails the case immediately with `agent error: <message>` and the expression is not evaluated. Referencing `error` or `status` in `expected_result` opts the case out of that short-circuit — the expression becomes the source of truth, so assertions like `status == "blocked"` or `status == "failed" and "timed out" in error` work as expected. `"x" in error` against a successful run (where `error` is `null`) evaluates to `false` rather than raising.

**Not allowed**: function/method calls (`json.loads(...)`, `output.strip()`, `len(...)`, `re.search(...)`), `lambda`, comprehensions, generators, imports, attribute names starting with `_`, calling anything (including `__import__`, dunder methods).

**Works:**
```yaml
expected_result: 'status == "completed" and "Invoice #" in output'
expected_result: 'status == "completed" and "invoice_number" in output and "total_amount" in output'
expected_result: 'status == "completed" and context["customer_id"] == 42'
expected_result: 'status == "blocked"'                                  # input guardrail tripped
expected_result: 'status == "failed" and "timed out" in error'          # specific failure mode
```

**Won't parse — needs the builder's `cleanup` instead:**
```yaml
# All of these will fail at parse time. Don't write expected_result like this.
expected_result: '"positive" in output.lower()'                     # method call
expected_result: 'json.loads(output)["sentiment"] == "positive"'    # function call + import
expected_result: |
  (lambda s: __import__("json").loads(s)["x"])(output) == 1         # lambda + import
expected_result: 'any(k in output for k in ["a", "b"])'             # generator
```

For anything that requires parsing the output, regex, schema validation, or cross-field checks, put the check in a builder `cleanup` function.

`expected_tool_calls` uses the same safe expression grammar with three bindings: `invocations` is the number of matching calls, `params` is one call's arguments, and `context` is the builder dict. A bare tool name means at least one call. Tool names match either the local function name or the qualified ref. Top-level `and` separates per-invocation `params` filters from `invocations` predicates over the filtered count; if an expression contains only params predicates, `invocations >= 1` is implied. Repeat the same tool in multiple list entries to require distinct argument patterns. Use `expected_tool_call_order` separately when relative order matters.

### Asserting on triggered agents

When the agent under test calls `trigger_agent` or `trigger_agent_at`, use `expected_child_agents` to assert on the triggered agent:

```yaml
tests:
  - name: dispatches_to_summarizer
    payload: '{"text": "..."}'
    expected_child_agents:
      summarizer:
        expected_payload: payload.text != ""
        expected_result: output.summary != ""
        expected_tool_calls:
          - llm.complete: invocations >= 1
        expected_no_tool_calls:
          - email.send

  # Pin the trigger payload against builder context — fails if the agent
  # forwards the wrong fixture id instead of the one it was given.
  - name: forwards_charge_id_unchanged
    builder: create_charge_then_refund
    expected_child_agents:
      billing-refunder:
        expected_payload: payload.charge_id == context.charge_id

  # Recursive: assert on a grandchild that summarizer triggers in turn.
  - name: dispatches_summarizer_then_publisher
    payload: '{"text": "..."}'
    expected_child_agents:
      summarizer:
        expected_result: output.summary != ""
        expected_child_agents:
          publisher:
            expected_tool_calls:
              - kafka.publish: params.topic == "summaries"

  # Fire-and-forget (wait_for_response=False) — output/tool calls aren't
  # observable, but the payload IS captured at call time, so
  # expected_payload + expected_triggered still apply.
  - name: fans_out_telemetry
    payload: '{"event": "checkout"}'
    expected_child_agents:
      telemetry-writer:
        expected_triggered: 1
        expected_payload: payload.event == "checkout"
```

Field shape — each entry under `expected_child_agents` takes:

- `expected_triggered: <int>` — minimum trigger count (default `1`).
- `expected_payload: <expr>` — expression over the input passed to `trigger_agent`. Bindings: `payload` (JSON-parsed when the parent passed a JSON string, else the raw value), `payload_raw` (the string form, `""` when N/A), `context` (the builder dict). Works on every trigger record regardless of mode.
- `expected_result`, `expected_tool_calls`, `expected_tool_call_order`, `expected_no_tool_calls` — same grammar as the top-level fields, evaluated against the child run. Require at least one `wait_for_response=True` trigger.
- `expected_child_agents` — recursive map for whatever this child triggers in turn.

Two evaluation paths:

- **`wait_for_response=True`** — result, tool, order, and nested assertions apply. When the same child is triggered more than once, the assertion passes as soon as one waited trigger satisfies the specification.
- **`wait_for_response=False`** — fire-and-forget. `expected_triggered` and `expected_payload` work; deeper assertions don't (the case fails with a clear reason telling you to wait for the response). `trigger_agent_at` is always treated as fire-and-forget in test mode.

The builder `context` dict is shared across every depth — a fixture id stashed in `build()` is reachable via `context.<key>` inside any child's `expected_payload`, `expected_result`, or `expected_tool_calls`.

### Builders — dynamic payloads, cleanup, and complex assertions

Builders live at `tests/builders/<name>.py`. Use them to generate the input payload programmatically or run post-run assertions that `expected_result` cannot express. Each test case loads the builder separately; `build` and `cleanup` may be sync or async and can use the agent's environment variables and network access. A missing builder fails before the agent runs. When a case also sets `files`, fixtures merge into the builder output; if a dict result already has a `files` list, the lists concatenate.

**`cleanup` contract** — Connic calls `cleanup(run, context, builder_args)` after the case completes. Its return value decides the result:

- `None` or `True` → the case passes
- `False` → the case fails with reason `"builder cleanup returned False"`
- Anything else (a string, an int, etc.) → the case fails with `"builder cleanup must return bool or None, got <type>"`
- An uncaught exception → the case fails with `"builder cleanup raised: <message>"` (works, but prefer returning `False` so you can shape the failure deliberately)

Return `False` early when an assertion doesn't hold. Use `print(...)` (captured in the run log) to leave a breadcrumb explaining why.

```python
# tests/builders/extract_invoice.py
import json
from typing import Any

REQUIRED_KEYS = {"invoice_number", "total_amount"}

def build(context: dict, builder_args: dict, test_name: str,
          payload: Any, files: list) -> dict:
    """Return the payload sent to the agent. Anything stashed on `context`
    here is threaded through to cleanup() for the same case."""
    return {"task": "extract this invoice"}

def cleanup(run, context: dict, builder_args: dict) -> bool:
    """Real Python — parse the output and check shape."""
    output = run["output"]

    # Strip ```json … ``` fences before parsing
    text = output.strip().strip("`")
    if text.startswith("json"):
        text = text[len("json"):].strip()

    try:
        parsed = json.loads(text)
    except json.JSONDecodeError as e:
        print(f"output is not valid JSON: {e}")
        return False

    missing = REQUIRED_KEYS - set(parsed)
    if missing:
        print(f"output missing required keys: {missing}")
        return False
    if not isinstance(parsed["total_amount"], (int, float)):
        print(f"total_amount must be numeric, got {type(parsed['total_amount']).__name__}")
        return False

    return True
```

```yaml
# tests/extract.yaml
tests:
  - name: invoice_has_required_fields
    builder: extract_invoice
    expected_result: status == "completed"     # cheap shape check
    # The real assertions live in extract_invoice.cleanup()
```

Use `expected_result` for status, tool-call, and substring checks. Use `cleanup` for parsing, schema validation, numeric ranges, and checks that need function calls.

### Testing approvals (HITL)

Use `approval_decisions` to supply responses to pending approvals:

```yaml
tests:
  - name: approves_the_exact_refund
    builder: create_charge_then_refund
    approval_decisions:
      - tool: billing.refund
        params: params.charge_id == context.charge_id
        decision: approve
        reason: Approved by this test
    expected_result: status == "completed"
```

- `tool` is the canonical tool ref. Optional `params` is a safe expression with `params`, builder `context`, `true`, `false`, and `null` bindings; omit it to match any parameters for that tool.
- `decision` is `approve`, `reject`, or `timeout`; `reason` is optional. Rejections and timeouts honor the approval's `on_rejection` setting, so they either terminate the run or resume it with rejection context.
- Each entry is consumed at most once per invocation.
- With `strict_approval_decisions: false` (the default), an unmatched pending approval returns `status == "awaiting_approval"`, and unused entries are ignored.
- Set `strict_approval_decisions: true` per case or in `defaults` to fail on unmatched pending approvals and unused entries.

### Mocking tools

Point a case at a `tests/mocks/<name>.py` module with the `mocks:` field to replace selected custom file tool results or lifecycle phases. A matching function replaces the real function for an existing phase; it does not add a missing phase. Without a match, the real project code runs by default. Custom guardrail files must still load successfully.

#### Tool result replacements

The module exposes hierarchical `mock_*` functions. For a tool ref like `data.customer.add_customer`, Connic uses the **most specific** one defined, trying in order:

| Function | Stands in for |
| --- | --- |
| `mock_data_customer_add_customer` | that exact tool |
| `mock_data_customer` | every tool in `tools/data/customer.py` |
| `mock_data` | every tool under `tools/data/` |
| `mock` | every custom file tool |

Every mock has the same signature and returns the substituted result:

```python
# tests/mocks/customer_mocks.py
def mock_data_customer_add_customer(tool_name, params, context):
    # tool_name -> the full ref ("data.customer.add_customer")
    # params    -> the args the agent passed to the tool
    # context   -> the run context dict
    return {"id": "cust_test_1", **params}
```

Only custom file tool implementations are eligible. Predefined tools (`db_find`, `web_search`, `trigger_agent`, …) and `api:` tool implementations always run for real.

#### Middleware replacements

Use the real middleware signatures and return contracts:

| Function | Replaces |
| --- | --- |
| `middleware_before(content, context) -> content` | `middleware/<agent>.py::before` |
| `middleware_after(response, context) -> response` | `middleware/<agent>.py::after` |

#### Tool hook replacements

Hook replacements use the same tool hierarchy with the phase appended. For `data.customer.add_customer`, before-hook resolution is:

1. `mock_data_customer_add_customer_hook_before`
2. `mock_data_customer_hook_before`
3. `mock_data_hook_before`
4. `mock_hook_before`

The after-hook ladder uses the same prefixes ending in `_hook_after`. Non-alphanumeric runs in each tool-ref segment normalize to `_`, so `api:weather-v2.lookup` resolves `mock_api_weather_v2_lookup_hook_before`. Before handlers have the signature `(tool_name, params, context) -> params`; after handlers use `(tool_name, params, result, context) -> result`.

Hook replacement is independent of tool-result replacement: either, both, or neither may match. A missing hook replacement runs the real hook by default; `strict_hook_mocks` fails before it executes instead. Replacements follow normal hook scope; remote MCP tools do not run agent hooks, so they never resolve hook replacements.

#### Custom guardrail replacements

Only `type: custom` guardrails are eligible. For an input guardrail named `domain_check`, resolution is:

1. `guardrail_input_domain_check`
2. `guardrail_input`
3. `guardrail`

Output guardrails use the equivalent `guardrail_output_<name>` → `guardrail_output` → `guardrail` ladder. Each handler mirrors the real `(content, context) -> GuardrailResult` contract. Non-alphanumeric runs in a guardrail name normalize to `_`, so `domain-check` resolves `guardrail_input_domain_check`. Built-in guardrails always run for real.

```yaml
# tests/customer-agent.yaml
defaults:
  strict_mocks: true
  strict_hook_mocks: true
  strict_middleware_mocks: true
  strict_guardrail_mocks: true       # every flag is also settable per-case
tests:
  - name: adds_without_touching_the_db
    payload: '{"name": "Ada"}'
    mocks: customer_mocks
    expected_tool_calls:
      - data.customer.add_customer: params.name == "Ada"
```

- **Tracing and assertions.** A mocked call appears in the trace (tagged `mocked` in the run drawer) and counts toward `expected_tool_calls` / `expected_no_tool_calls`.
- **Parameter validation.** Mocked arguments are validated against the real tool's signature (required arguments, types, and unknown arguments), so a malformed call fails. Defaulted parameters are optional.
- **`strict_mocks` is tool-only.** Set it per case or in `defaults` to abort before an unmocked custom file tool executes. It does not govern middleware, hook, or guardrail replacements. Predefined and `api:` tools are exempt.
- **Lifecycle strictness is independent.** `strict_hook_mocks`, `strict_middleware_mocks`, and `strict_guardrail_mocks` each default to `false` and can be set per case or in `defaults`, independently of `strict_mocks` and one another. Each aborts before a configured eligible real phase of that kind executes without a matching replacement. Hook and middleware phases that are not configured are exempt, as are missing guardrail phases and all built-in guardrails.
- Mock module state resets for each invocation. A typo in `mocks:` fails before the agent runs.

Flags:

- `--env <environment-id>` — run the tests against a specific environment (defaults to the env's `test_environment_id`, falling back to itself).
- `--filter <substring>` — run only tests whose names contain the substring (there is no `--grep`).
- `--coverage` — static no-network analysis of which agents and tools your tests touch.
- `--json` — emit the test (or coverage) report as JSON for tooling/CI. Coverage JSON is shaped like `{overall, agents: [{name, type, has_tests, tools_total, tools_covered, uncovered_tools, percent, parse_error}]}`; fail CI on `parse_error` if any local suite cannot be parsed.

Test exit codes are `0` when every case passes, `1` for a failed or cancelled test run or a CLI request error, and `2` for an infrastructure or server error. Coverage is an equal-weight average of agent percentages, not one project-wide tool ratio. `expected_tool_calls` and `expected_tool_call_order` both count; `discoverable_tools` are included in the denominator; a tested tool-less agent scores 100%; agents without suites score 0%; and A/B test variants are excluded.

## `connic deploy`

```bash
connic deploy                              # deploys to the project's default environment
connic deploy --env=<environment-uuid>     # target a specific env (UUID, not name)
connic deploy --skip-tests                 # bypass the test gate (hotfix only)
```

There is no `--message` / `-m` flag. The `--env` value is an **environment UUID** (copy from the dashboard), not the human-readable name.

**`connic deploy` refuses to run on a project that has a connected Git repo.** Use `git push` to the configured branch in that case — that triggers the same pipeline (build → tests → deploy). The CLI deploy is for projects without Git integration.

Failing tests gate deploys. Use `--skip-tests` only as an escape hatch; Git-triggered deploys cannot skip tests.

## `connic migrate`

```bash
connic migrate
connic migrate --source ./old-langchain-project --dest ./new-connic-project
connic migrate --source ./adk-project --dest ./new-connic-project
```

With no flags, the CLI prompts for the source and destination paths; if one flag is omitted, it prompts only for the missing path. Source and destination cannot be the same, nested inside one another, or an already non-empty destination. There is no positional path argument and no `--output` flag. The tool emits a Connic project skeleton, runs `connic lint`, and writes `MIGRATION_REPORT.md`.

- **LangChain:** automatically migrates top-level `create_agent` / `create_react_agent` calls, plain or `@tool` functions and their local dependencies, static prompts, common model constructors, and cross-file imports. LangGraph state graphs and handoffs, dynamic prompts, checkpointers, custom persistence, RAG chains, LangSmith integration, wrappers/subclasses, and non-top-level agents remain manual.
- **Google ADK:** automatically migrates explicit `Agent`, `LlmAgent`, and `SequentialAgent` definitions, plain functions / `FunctionTool`, `AgentTool` to `trigger_agent`, static instructions, model names, YAML agents, and `google_search` to `web_search`. `ParallelAgent` / `LoopAgent` become sequential placeholders; callbacks, sessions/memory, MCPToolset, dynamic registration/instructions/tool lists, subclasses, and custom guardrails need manual work.

Always inspect the report and generated code: migration is a starting point, not a finished port.

## Iteration loop (recommended)

```
1. connic init my-project --skill && cd my-project
2. connic login
3. Edit agents/, tools/, middleware/ ...
4. connic lint                       # local check
5. connic dev                        # cloud hot-reload (trigger via webhook / dashboard "Run")
6. connic test                       # validate
7. git push                          # auto-deploy on Git-connected projects
   # OR (only if not Git-connected): connic deploy --env=<uuid>
```

## When a flag isn't in this doc

`connic <command> --help` is the canonical list. Anything beyond that — invented flags like `--grep` (it's `--filter` on `test`), `--message` on `deploy`, `--json` on `lint` (only `connic test` has `--json`), `connic dev --env`, `connic logs`, `connic build`, `connic run` — does not exist.
