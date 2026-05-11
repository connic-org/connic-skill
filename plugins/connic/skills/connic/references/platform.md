# Platform (Dashboard) concepts

The dashboard at `connic.co` is where projects, environments, deployments, and operational concerns live. Most of these aren't expressed in the on-disk YAML — they're properties of the running project on Connic's infrastructure.

## Project

The top-level container in Connic. Every dashboard URL is scoped to a project. **Always call it a "project", never "workspace".**

A project has: a set of environments, its own API keys, its own connectors, its own Git connection, its own team membership. The knowledge base and the database are **per-environment**, not per-project (so staging and production have separate data).

## Environments

Each project has multiple environments (e.g. `staging`, `production`), plus a separate pool of **dev environments** that back `connic dev` sessions. Environments are fully isolated:

- Separate environment variables / secrets.
- Separate connector configurations.
- Separate database and knowledge base data.
- Separate API keys for triggering agents externally.

Configure under **Project Settings → Environments**. Map a Git branch to each environment for auto-deploy on push. Each environment also has an optional "Test environment" pointer — when set, the deploy gate runs `connic test` against that environment instead of the target, so production deploys can validate with stub credentials.

## Deployment

Two paths:

1. **Git-connected (preferred).** Configure a GitHub/GitLab/Bitbucket repo in **Project Settings → Git**. Each branch can map to one environment. Pushing runs a three-step pipeline: **Build → Tests → Deploy**. A failing test blocks promotion; git-triggered deploys cannot skip tests.
2. **CLI.** `connic deploy --env=<env-uuid>` from a directory with a valid `.connic`. The `--env` value is an environment UUID, not the human name. `connic deploy` refuses to run if the project has a Git connection — Git is single-source-of-truth in that case. `--skip-tests` is available as a breaking-glass option for non-Git deploys.

A deploy bundles the local files (agents, tools, middleware, hooks, schemas, guardrails, tests, requirements.txt) and uploads them to Connic. The new bundle becomes the active version for that environment. Failed builds never replace the live deployment, and previous deployments remain available via an "Activate" action for instant rollback.

Environment variables are injected at deploy time. Changing a variable requires re-deploying for new runs to pick it up.

## Observability

**Dashboard → Logs** and **Dashboard → Runs** show every agent invocation:

- Inputs, outputs, every tool call with args and result.
- Captured `print()` and `logging` calls. Python logger names must start with `tools.`, `middleware.`, `hooks.`, or `guardrails.` — the dashboard surfaces them as "Tool", "Middleware" (tagged `before`/`after`), "Hook", or "Guardrail" entries.
- Token usage, latency, cost per run.
- Reasoning traces (if `reasoning: true` in the agent).
- Unhandled exceptions are captured automatically with their tracebacks.

There's a 500-log-lines-per-run cap. Logs view filters include Status, Date Range, Deployment, and Search.

## Knowledge base

**Per-environment** vector store. Used via the predefined tools `query_knowledge`, `store_knowledge`, `delete_knowledge`, `kb_list_namespaces` (see [predefined-tools.md](predefined-tools.md)).

Dashboard view: **Project → Knowledge Base** (scoped to the active environment). Inspect entries, edit content, drop namespaces, bulk-import via CSV or file upload.

Namespaces are **dot-separated**, hierarchical, max depth 10 — e.g. `policies.hr.leave`, `products.pricing`. Don't use slashes. Ingestion is asynchronous: `store_knowledge` returns a job ID and entries only become searchable once indexing completes.

## Database

**Per-environment** schemaless document store. Used via `db_find` / `db_insert` / `db_update` / `db_delete` / `db_count` (see [predefined-tools.md](predefined-tools.md)).

Dashboard view: **Project → Database**. Browse collections, run ad-hoc queries, inspect documents, export.

Production data is never visible from staging.

## Judges

Automated evaluators that score runs after the fact (correctness, helpfulness, safety, custom criteria). Configure in **Project → Judges**. A judge is itself an LLM evaluator that takes a completed run as input and emits a score + reason.

- Judges are **per-agent only** — set one up against the specific agent you want graded.
- Triggers: automatic on every run, automatic on a sample (configurable rate), or manual.
- Filter expressions over `context.*` let you grade only matching runs.
- Score-alert thresholds with a rolling window send notifications when quality drops.

## Approvals

Human-in-the-loop gating for specific tool calls. Configured in the agent YAML (`approval:` block — see [agent-yaml.md](agent-yaml.md)). When an agent hits a gated tool, the run pauses and appears in **Project → Approvals**. A reviewer approves or rejects; the run resumes (or fails / skips the tool depending on `on_rejection`). Conditions on `param.*` and `context.*` let you gate only some calls. An approval webhook can POST decisions to your own endpoint.

## A/B testing

Compare agent configurations side-by-side. Variants are declared as YAML files following the `{base-agent}-test-{variant}.yaml` naming convention (the loader auto-recognises them — see [agent-yaml.md](agent-yaml.md#ab-test-variants)). Configure traffic percentages and metrics in **Project → A/B Testing**. Multiple concurrent tests are supported as long as total traffic across them is ≤100%. Judges typically supply the win-rate metric.

## Bridge

A tunnel from Connic's cloud to a private network (your VPC, on-prem services). Provision a Bridge under **Project Settings → Bridge**, install the Bridge agent on a machine inside your network. The Bridge ID can then be referenced from four kinds of consumer:

1. **Connectors** — pick the Bridge in the connector config dropdown.
2. **Custom LLM providers** — pick the Bridge in the provider config.
3. **Custom tools / middleware** — reach private endpoints via the magic hostname `<target>.cnc-bridge-<bridge_id>` from inside your Python code.
4. **MCP servers** — set the `bridge:` field on the `mcp_servers` entry.

```yaml
mcp_servers:
  - name: internal
    url: http://mcp.internal:8080/mcp
    bridge: ${INTERNAL_BRIDGE_ID}
```

```python
# From a custom tool reaching a private internal API
import httpx
url = f"https://api.internal.cnc-bridge-{os.environ['INTERNAL_BRIDGE_ID']}/users"
r = await httpx.AsyncClient().get(url)
```

## Domains

Bring your own subdomain for inbound connectors (custom webhook URL, custom MCP server endpoint). Configure DNS records as shown in **Project Settings → Domains**; Connic provisions TLS automatically.

Restrictions: **subdomains only** (no apex/wildcard); requires the Pro plan or higher; only Owners and Admins can manage domains.

## Team

**Project Settings → Team** manages member roles. The three roles are **Owner**, **Admin**, and **Member** — there are no "Developer" or "Viewer" roles, and roles are not scoped per-environment.

For finer-grained access, use **API keys** instead — keys are scoped per section (`agents`, `runs`, `knowledge`, `judges`, `budgets`, `audit-logs`, `deployments`) with read/write granularity.

The audit log (under Team) records actions across categories: project, deployment, connector, environment, variable, API key, billing, member, invite, Git.

> **Team roles ≠ end-user authentication.** Owner/Admin/Member governs **who can sign in to the dashboard and manage the project**. It is not a model for authenticating the end users whose requests trigger agents. End-user auth (JWTs, OIDC tokens, per-user permissions) belongs in `middleware/<agent>.py::before`, with conditional tools gating actions on the hydrated `context.permissions`. See the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions).

## Usage

**Project → Usage** breaks down LLM token spend and run counts by agent, environment, and period. Budget controls (hard limits, alerts, anomaly detection, scheduled reports) live in the same section.

Architectural recommendations in this skill should be made on the basis of fit, reliability, and maintainability — not cost. Use the dashboard's usage and budget tools to manage spend separately from design decisions.

## REST API

`https://api.connic.co/v1/...`. Auth via API key (create under **Project Settings → CLI / API Keys**). Rate limit: 60 requests/minute per project — 429 responses include `Retry-After`.

The REST API is for **admin and tooling work**: listing runs, reading logs, managing connectors, managing deployments, pulling usage data, managing knowledge entries. **It is not the path for triggering agents from external callers** — for that you create a connector.

If a user is wiring an external service to a Connic agent and reaches for the REST API, redirect them: the right answer is a **`webhook` connector** (or `kafka`, `sqs`, `email`, `telegram`, etc. depending on the transport). Connectors give you per-agent URLs with their own secrets, signed payloads, sync vs async modes, replay safety, and a clear audit trail — none of which the REST API provides as a first-class feature. The platform models "things that trigger agents" as connectors on purpose; routing inbound traffic through the REST API bypasses the model.

For the full endpoint catalogue and per-section permission scopes, fetch `https://connic.co/docs/v1/platform/rest-api`.

## What lives where: dashboard vs. on-disk

| Concern | Lives in… |
| --- | --- |
| Agent definition (prompt, tools, model, etc.) | On-disk YAML (`agents/`) |
| Tool implementations | On-disk Python (`tools/`) |
| Middleware / hooks / guardrails | On-disk Python |
| JSON output schemas | On-disk (`schemas/`) |
| Declarative tests | On-disk (`tests/`) |
| Environment variables / secrets | Dashboard (per environment) |
| Connector configuration | Dashboard (linked to agents) |
| API spec imports | Dashboard |
| Knowledge base content | Dashboard or via `store_knowledge` tool |
| Database collections | Dashboard or via `db_*` tools |
| Judges, A/B tests, approvals queue | Dashboard |
| Team members and roles | Dashboard |
| Deployment & branch mapping | Dashboard (Git settings) |
| Bridge tunnels and custom domains | Dashboard |

Keep this split clear when answering "where do I configure X?" — connectors and secrets are common confusion points and they're dashboard-only.
