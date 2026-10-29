# Project anatomy

A connic project is a directory of YAML and Python files. The CLI (`connic` from `connic-composer-sdk`) reads this directory, validates it, and syncs it to connic cloud.

## Canonical layout

```
my-project/
├── .connic                    # api_key + project_id (DO NOT commit)
├── .agents/skills/connic/     # optional — installed by `connic skill` or `connic init --skill`
├── .claude/skills/connic/     # optional — installed alongside the .agents copy
├── requirements.txt           # extra Python deps (SDK installed globally)
├── README.md
├── agents/                    # *.yaml — agent definitions (nesting allowed)
│   ├── _defaults.yaml         # optional — defaults inherited by every agent below
│   ├── support-assistant.yaml
│   └── examples/
│       ├── _defaults.yaml     # optional — adds/overrides defaults for this subtree
│       └── foo.yaml           # nested OK
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
├── connectors/                # *.yaml connectors, applied by deploys (flat, no nesting)
│   └── order-events.yaml
├── apps/                      # static frontends, built and served by deploys
│   ├── portal.yaml            # declares the app; the file name is its key
│   └── portal/                # its sources: package.json, lockfile, src/
└── tests/                     # *.yaml — declarative test suites
    ├── support-assistant.yaml
    ├── files/                  # binary fixtures referenced by files:
    ├── builders/               # *.py — dynamic-payload builders referenced by builder:
    └── mocks/                  # *.py — custom-code replacements referenced by mocks:
```

`connic init` creates `agents/`, `tools/`, `middleware/`, `schemas/`, plus `.gitignore`, `requirements.txt`, and a `README.md` containing an example agent. It does **not** create a `.connic` file (that's `connic login`'s job) and it does **not** drop a stub agent on disk. Add `--skill` to install the connic skill under `.agents/skills/connic/` and `.claude/skills/connic/`. Add `hooks/`, `guardrails/`, `tests/`, `connectors/`, `apps/` as you need them.

## Auto-discovery rules

These are how files wire up. There is no central manifest — the layout *is* the manifest.

| Convention | What it means |
| --- | --- |
| `agents/<name>.yaml` with `name: <name>` inside | Defines an agent. Nesting under `agents/` is fine; the file basename doesn't have to match `name`. |
| `agents/**/_defaults.yaml` | Defaults inherited by every agent at that directory level and below. See [Cascading `_defaults.yaml`](#cascading-_defaultsyaml) and the [agent-yaml reference](agent-yaml.md#cascading-defaults-with-_defaultsyaml). |
| `middleware/<agent-name>.py` | Auto-attached to the agent with that `name`. Define any of `async def before(content, context)`, `async def after(response, context)`, and `async def cleanup(run, context)` (runs once the run ends, whatever the outcome). |
| `hooks/<agent-name>.py` | Auto-attached. Define `async def before(tool_name, params, context)` and/or `async def after(tool_name, params, result, context)`. Both signatures may omit the `context` parameter. |
| `tools/<module>.py` with function `foo` | Reference as `<module>.foo` in an agent's `tools:` list. Nested: `tools/math/calc.py::add` → `math.calc.add`. Wildcards (`module.*`, `module.search_*`) supported. |
| `schemas/<name>.json` | Reference as `output_schema: <name>` in an agent. Valid JSON Schema (subset — see [guardrails-schemas-mcp.md](guardrails-schemas-mcp.md#output-schemas)). |
| `guardrails/<name>.py` | Reference as `name: <name>` inside a `guardrails: - type: custom` block — filename must match exactly. |
| `connectors/<key>.yaml` | Declares one connector; the file name is its key and identity. Deploys create, update, or retire it per environment. See [Connectors in code](connectors.md#connectors-in-code). |
| `apps/<key>.yaml` with `apps/<key>/` | Declares one app, built from `apps/<key>/` and served on its own URL; the file name is its key. See [Apps](apps.md). |

connic skips `_`-prefixed **files** and **functions** during tool discovery. So `tools/_helpers.py` is not exposed, and neither is `tools/utils.py::_internal_fn`. Use the leading underscore for module-private helpers. `__init__.py` is one example of a `_`-prefixed file; do not put tools there.

Parameters without type hints are still exposed but default to `{"type": "string"}` in the LLM-facing schema, which gives the model the wrong guidance for numeric or boolean values. Type-hint every parameter. Missing docstrings show up as `"No description provided."` to the LLM — write docstrings.

## Cascading `_defaults.yaml`

At any directory under `agents/`, drop a `_defaults.yaml` to share configuration with every agent at that level and deeper. connic combines `_defaults.yaml` files shallowest-first; the agent's own YAML wins last.

```
agents/
├── _defaults.yaml              # applies to every agent in the project
├── process/
│   ├── _defaults.yaml          # adds/overrides for everything under process/
│   ├── ingest/
│   │   └── foo.yaml            # inherits root + process defaults
│   └── enrich/
│       ├── _defaults.yaml      # adds/overrides for process/enrich/*
│       └── bar.yaml            # inherits root + process + enrich defaults
└── support-assistant.yaml      # inherits only root defaults
```

`_defaults.yaml` follows the same shape as a normal agent YAML, but is partial — only the fields you want to share. `name` and `description` are **forbidden** in defaults because they identify one agent. `version` is permitted, but each agent file must also set `version`, `name`, and `description`.

For the precise merge rules (scalars override, dicts deep-merge, lists concat with dedup-by-ref), see [Cascading defaults with `_defaults.yaml`](agent-yaml.md#cascading-defaults-with-_defaultsyaml) in the agent YAML reference.

## The `.connic` file

Credentials live here. Created by `connic login`. **Do not commit.** Add it to `.gitignore`.

```json
{
  "api_key": "cnc_xxxxxxxxxxxx",
  "project_id": "uuid-here"
}
```

## requirements.txt

List only project-specific Python dependencies such as HTTP clients and database drivers. connic provides `connic-composer-sdk`; do not add it here.

Example:

```
httpx>=0.25.0
asyncpg>=0.29.0
aiosmtplib>=3.0.0
```

If the project has no extra deps, the file can be empty or omitted.

Each line must be a plain requirement specifier; options (`-r`, `--index-url`), URLs and file paths are rejected. Packages install only from prebuilt wheels for Python 3.12 on Linux x86-64; source builds are disabled. If a package or any of its dependencies is published only as an sdist, the build fails. Check before deploying with `uv pip compile requirements.txt --only-binary :all: --python-version 3.12 --python-platform x86_64-manylinux_2_28`.

## What about `pyproject.toml` / `setup.py`?

connic deployment ignores Python packaging metadata. A larger repository may still use `pyproject.toml` or `setup.py` for its own tooling, but connic installs runtime dependencies only from the project root's `requirements.txt`; do not rely on package metadata to deploy them.

## Naming conventions

- **Agent name** (and file basename): kebab-case (`support-assistant`, `invoice-extractor`).
- **Python module / file**: snake_case (`billing_tools.py`).
- **Function names**: snake_case.
- **Schema file**: kebab-case to match agent style (`invoice-data.json`).

Use the project's existing module and schema style when it differs. Agent names must still use lowercase letters, numbers, and hyphens.

## Multiple agents sharing tools

Tools in `tools/` are global to the project. Any agent can reference any tool by its module path. There's no per-agent namespacing — if you want a tool to be available only to one agent, name it specifically and just don't reference it from others.

## When you're editing a project

Before changing or adding any file, scan the existing structure (`ls agents/ tools/ middleware/`) to learn the project's chosen conventions. If the project uses snake_case agent names, follow that; if existing tools are organized into subdirectories by domain, mirror that organization.
