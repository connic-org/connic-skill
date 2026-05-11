# Project anatomy

A Connic project is a directory of YAML and Python files. The CLI (`connic` from `connic-composer-sdk`) reads this directory, validates it, and syncs it to Connic cloud.

## Canonical layout

```
my-project/
├── .connic                    # api_key + project_id (DO NOT commit)
├── requirements.txt           # extra Python deps (SDK installed globally)
├── README.md
├── agents/                    # *.yaml — agent definitions (nesting allowed)
│   ├── support-assistant.yaml
│   └── examples/foo.yaml      # nested OK
├── tools/                     # *.py — plain function modules
│   ├── billing.py
│   └── math/calculator.py     # nested OK
├── middleware/                # *.py — basename matches agent name
│   └── support-assistant.py
├── hooks/                     # *.py — basename matches agent name
│   └── support-assistant.py
├── schemas/                   # *.json — JSON Schema output validators
│   └── invoice.json
├── guardrails/                # *.py — custom guardrail modules
│   └── domain_check.py
└── tests/                     # *.yaml — declarative test suites
    └── support-assistant.yaml
```

`connic init` creates `agents/`, `tools/`, `middleware/`, `schemas/`, plus `.gitignore`, `requirements.txt`, and a `README.md` containing an example agent. It does **not** create a `.connic` file (that's `connic login`'s job) and it does **not** drop a stub agent on disk. Add `hooks/`, `guardrails/`, `tests/` as you need them.

## Auto-discovery rules

These are how files wire up. There is no central manifest — the layout *is* the manifest.

| Convention | What it means |
| --- | --- |
| `agents/<name>.yaml` with `name: <name>` inside | Defines an agent. Nesting under `agents/` is fine; the file basename doesn't have to match `name`. |
| `middleware/<agent-name>.py` | Auto-attached to the agent with that `name`. Define `async def before(content, context)` and/or `async def after(response, context)`. |
| `hooks/<agent-name>.py` | Auto-attached. Define `async def before(tool_name, params, context)` and/or `async def after(tool_name, params, result, context)`. Both signatures may omit the `context` parameter. |
| `tools/<module>.py` with function `foo` | Reference as `<module>.foo` in an agent's `tools:` list. Nested: `tools/math/calc.py::add` → `math.calc.add`. Wildcards (`module.*`, `module.search_*`) supported. |
| `schemas/<name>.json` | Reference as `output_schema: <name>` in an agent. Valid JSON Schema (subset — see [guardrails-schemas-mcp.md](guardrails-schemas-mcp.md#output-schemas)). |
| `guardrails/<name>.py` | Reference as `name: <name>` inside a `guardrails: - type: custom` block — filename must match exactly. |

`_`-prefixed **files** and **functions** are skipped by the loader. So `tools/_helpers.py` is invisible, and so is `tools/utils.py::_internal_fn`. Use the leading underscore to hide module-private helpers. `__init__.py` is just one example of a `_`-prefixed file — don't put public tools there.

Parameters without type hints are still exposed but default to `{"type": "string"}` in the LLM-facing schema, which silently coerces numeric/boolean inputs. Type-hint everything. Missing docstrings show up as `"No description provided."` to the LLM — write docstrings.

## The `.connic` file

Credentials live here. Created by `connic login`. **Do not commit.** Add it to `.gitignore`.

```json
{
  "api_key": "cnc_xxxxxxxxxxxx",
  "project_id": "uuid-here"
}
```

## requirements.txt

List only project-specific Python deps (HTTP clients, DB drivers, etc.). The `connic-composer-sdk` is installed by the runner, not from this file.

Example:

```
httpx>=0.25.0
asyncpg>=0.29.0
aiosmtplib>=3.0.0
```

If the project has no extra deps, the file can be empty or omitted.

## What about `pyproject.toml` / `setup.py`?

A Connic project is *not* a Python package. Don't create one. The runtime imports modules by path; packaging machinery interferes with discovery.

## Naming conventions

- **Agent name** (and file basename): kebab-case (`support-assistant`, `invoice-extractor`).
- **Python module / file**: snake_case (`billing_tools.py`).
- **Function names**: snake_case.
- **Schema file**: kebab-case to match agent style (`invoice-data.json`).

Mixing styles works but is hard to read. Stick to the above unless the user's repo already uses something else.

## Multiple agents sharing tools

Tools in `tools/` are global to the project. Any agent can reference any tool by its module path. There's no per-agent namespacing — if you want a tool to be available only to one agent, name it specifically and just don't reference it from others.

## When you're editing a project

Before changing or adding any file, scan the existing structure (`ls agents/ tools/ middleware/`) to learn the project's chosen conventions. If the project uses snake_case agent names, follow that; if existing tools are organized into subdirectories by domain, mirror that organization.
