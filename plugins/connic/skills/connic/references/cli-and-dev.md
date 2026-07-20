# The `connic` CLI and dev workflow

The CLI ships with `connic-composer-sdk`. Install with `pip install connic-composer-sdk` (requires Python 3.10+).

## Full command list

| Command | What it does |
| --- | --- |
| `connic init [name]` | Scaffold a new project directory. `--templates=invoice,customer-support` seeds from starter templates. |
| `connic login` | Browser-based auth; writes `.connic` (api_key + project_id) into the current directory. `--token <project_id>:<api_key>` skips the browser for CI. |
| `connic lint` | Validate YAML, tool references, schemas, middleware/hooks discovery — locally, no upload. |
| `connic tools` | List every tool the project exposes, with a one-line description and the source module path. |
| `connic dev [name]` | Open a cloud dev environment, sync local files, hot-reload on save. Named sessions persist; unnamed are ephemeral. |
| `connic test` | Run declarative test suites from `tests/` against an environment. `--env <id>` picks the environment; `--filter <substring>` runs a subset; `--coverage` runs a static no-network coverage report. |
| `connic deploy` | Deploy current files to a Connic environment. Refuses to run if the project is connected to a Git repo (use `git push` in that case). |
| `connic migrate --source <path> --dest <path>` | Scan a LangChain or Google ADK project and emit a Connic-shaped project skeleton. |

Run any with `--help` for the canonical flag list.

## `connic init`

```bash
connic init my-agents
cd my-agents
connic init my-agents --templates=invoice,customer-support   # seed with examples
```

The default scaffold creates `agents/`, `tools/`, `middleware/`, `schemas/`, plus `.gitignore`, `requirements.txt`, and a `README.md` containing a starter example. It does **not** create a `.connic` file (that's `connic login`'s job) and it does **not** drop a stub agent YAML on disk — the README shows what one should look like.

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
- Missing required agent fields (`version`, `name`, `description`, `model` for LLM agents).
- Tool references that don't resolve (`tools: [billing.missing_function]`).
- Duplicate agent names across files.
- Schema files referenced by `output_schema:` that don't exist or aren't valid JSON.
- Middleware/hooks/guardrails modules that fail to import.

Run this before every deploy. There is no `--json` flag — `lint` only takes `--verbose` / `-v`.

## `connic tools`

```bash
connic tools
```

Prints every tool the runtime would expose, grouped by `tools/` module (each with a one-line description, no signatures). Use this to confirm:

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

- Spins up an isolated cloud runner with the same image production uses.
- Syncs `agents/`, `tools/`, `middleware/`, `hooks/`, `schemas/`, `guardrails/`, `tests/`, and `requirements.txt`. Changes to `requirements.txt` trigger a re-install on the next sync — no restart needed. `tests/` is synced too, so you can press `t` in the dev session to run the suites against the live runner.
- Watches files; resync in ~2–5 seconds.
- The dev session has its own variables, its own ephemeral database, and a fresh knowledge base — separated from your standard environments.

The `.connic` file is **not** synced — it's local auth only.

`connic dev` does not accept an `--env` flag. If you need to test against staging's data, point your dev session at staging credentials by other means (e.g. duplicate the relevant env vars into the dev session in the dashboard).

## `connic test`

Run all suites in `tests/` against the configured environment.

```yaml
# tests/sentiment.yaml
version: "1.0"

defaults:
  runs: 5                  # run each case 5 times
  success_threshold: 80    # % of runs that must pass
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

  - name: extract_invoice
    files:
      - invoice.pdf        # resolved from tests/files/invoice.pdf
    expected_result: '"Invoice #" in output'
```

Available assertion fields (these are the only ones):

- `expected_result` — an expression evaluated against the bindings `output`, `error`, `status`, `context` (plus `true` / `false` / `null`). See "What `expected_result` can and cannot do" below.
- `expected_tool_calls` — list of `tool_name: <expr>` pairs evaluated against `invocations` (the call count for that tool in the run).
- `expected_no_tool_calls` — list of tool names that must not be called.
- `expected_child_agents` — map of triggered agent name → assertions for that child run (`expected_payload`, `expected_result`, `expected_tool_calls`, `expected_no_tool_calls`, `expected_triggered`, plus a nested `expected_child_agents`). See "Asserting on triggered agents" below.

There are no `expected_output_contains` / `expected_output_matches` fields. Don't invent them.

Fixtures live in `tests/files/`. Reference them with `files: [<bare-filename>, ...]` (plural — never `file:`).

If `payload` is a JSON object (or comes from a `builder` that returns a dict), its keys sit at the top level of `context["payload"]`. Otherwise the string is delivered as `{message: <payload>}`. Attached `files` are added alongside, under a `files` list.

Per-case overrides for `runs`, `success_threshold`, and `timeout_s` are allowed.

Case execution controls (not assertions) include `mocks` (a `tests/mocks/<name>.py` module that can replace custom file tools and lifecycle phases), the independent `strict_mocks`, `strict_hook_mocks`, `strict_middleware_mocks`, and `strict_guardrail_mocks` booleans (all also settable in `defaults`), `approval_decisions` (scripted HITL responses), and `strict_approval_decisions` (bool, also settable in `defaults`). See "Testing approvals" and "Mocking tools" below.

### What `expected_result` can and cannot do

`expected_result` is **not** ordinary Python — it's a sandboxed expression evaluator with a very narrow AST surface. Use it for cheap shape checks; fall back to the builder's `cleanup` for anything more.

**Allowed**: boolean ops (`and`, `or`, `not`), comparisons (`==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `not in`, `is`, `is not`), subscripts (`output[...]`, `context["x"]`), attribute access (no underscore attrs — `output.id` works on a dict output and is equivalent to `output["id"]`), list/tuple/set/dict literals, and the names `output`, `error`, `status`, `context`, `true`, `false`, `null`.

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

For anything that requires parsing the output (JSON, regex, schema validation, cross-field checks), put the check in a **builder `cleanup`** — that's ordinary Python and can do whatever you need.

### Asserting on triggered agents

When the agent under test calls `trigger_agent` (or `trigger_agent_at`), the deploy-gate container dispatches the child agent **in-process** instead of hitting the live deployment. That gives the testing framework a real, captured run for every triggered agent, so you can assert on it with `expected_child_agents`:

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
- `expected_result`, `expected_tool_calls`, `expected_no_tool_calls` — same grammar as the top-level fields, evaluated against the child run. Require at least one `wait_for_response=True` trigger.
- `expected_child_agents` — recursive map for whatever this child triggers in turn.

Two evaluation paths:

- **`wait_for_response=True`** — the child runs synchronously in the test container with its own tool-call collector, so result/tool/nested assertions all apply. When the same child was triggered more than once, the assertion passes as soon as one waited trigger satisfies the spec.
- **`wait_for_response=False`** — fire-and-forget. `expected_triggered` and `expected_payload` work; deeper assertions don't (the case fails with a clear reason telling you to wait for the response). `trigger_agent_at` is always treated as fire-and-forget in test mode.

The builder `context` dict is shared across every depth — a fixture id stashed in `build()` is reachable via `context.<key>` inside any child's `expected_payload`, `expected_result`, or `expected_tool_calls`.

In-process dispatch is exclusive to the deploy-gate container. Production `trigger_agent` calls still route through the API path.

### Builders — dynamic payloads, cleanup, and complex assertions

Builders live at `tests/builders/<name>.py`. Use them for two distinct reasons: (a) generating the input payload programmatically, and (b) running arbitrary post-run assertions that `expected_result` can't express.

**`cleanup` contract** — the runtime calls `cleanup(run, context, builder_args)` after the case completes. Its return value decides the result:

- `None` or `True` → the case passes
- `False` → the case fails with reason `"builder cleanup returned False"`
- Anything else (a string, an int, etc.) → the case fails with `"builder cleanup must return bool or None, got <type>"`
- An uncaught exception → the case fails with `"builder cleanup raised: <message>"` (works, but prefer returning `False` so you can shape the failure deliberately)

Return `False` early when an assertion doesn't hold. Use `print(...)` (captured in the run log) to leave a breadcrumb explaining why.

```python
# tests/builders/extract_invoice.py
import json

REQUIRED_KEYS = {"invoice_number", "total_amount"}

def build(context: dict, builder_args: dict, test_name: str,
          payload: any, files: list) -> dict:
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

The split: `expected_result` for the cheap "did it finish, did it call the right tool, does the substring appear" checks; `cleanup` for parsing, schema validation, numeric ranges, anything that needs a function call. Don't try to cram complex logic into `expected_result` — it physically won't run.

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

The module exposes hierarchical `mock_*` functions. For a tool ref like `data.customer.add_customer`, the runner uses the **most specific** one defined, trying in order:

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
- One fresh re-import per invocation, like builders — module-level state resets between runs. A typo in `mocks:` fails fast, before the test container starts.

Flags:

- `--env <environment-id>` — run the tests against a specific environment (defaults to the env's `test_environment_id`, falling back to itself).
- `--filter <substring>` — run only tests whose names contain the substring (there is no `--grep`).
- `--coverage` — static no-network analysis of which agents and tools your tests touch.
- `--json` — emit the test (or coverage) report as JSON for tooling/CI. Coverage JSON is shaped like `{overall, agents: [{name, type, has_tests, tools_total, tools_covered, uncovered_tools, percent, parse_error}]}`; fail CI on `parse_error` if any local suite cannot be parsed.

## `connic deploy`

```bash
connic deploy                              # deploys to the project's default environment
connic deploy --env=<environment-uuid>     # target a specific env (UUID, not name)
connic deploy --skip-tests                 # bypass the test gate (hotfix only)
```

There is no `--message` / `-m` flag. The `--env` value is an **environment UUID** (copy from the dashboard), not the human-readable name.

**`connic deploy` refuses to run on a project that has a connected Git repo.** Use `git push` to the configured branch in that case — that triggers the same pipeline (build → tests → deploy). The CLI deploy is for projects without Git integration, or for breaking-glass scenarios where you've temporarily disconnected the repo.

Failing tests gate deploys. Use `--skip-tests` only as an escape hatch; Git-triggered deploys cannot skip tests.

## `connic migrate`

```bash
connic migrate --source ./old-langchain-project --dest ./new-connic-project
connic migrate --source ./adk-project --dest ./new-connic-project
```

Both `--source` and `--dest` are required. There is no positional path argument and no `--output` flag. The tool scans the source, identifies agents, tools, and prompts, and emits a Connic project skeleton at `--dest`. Always inspect the output — it's a starting point, not a finished port.

## Iteration loop (recommended)

```
1. connic init my-project && cd my-project
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
