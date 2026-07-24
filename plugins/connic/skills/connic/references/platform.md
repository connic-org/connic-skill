# Platform (Dashboard) concepts

The dashboard at `connic.co` is where projects, environments, deployments, and operational concerns live. Most of these aren't expressed in the on-disk YAML — they're properties of the running project on Connic's infrastructure.

## Project

The top-level container in Connic. Every dashboard URL is scoped to a project. **Always call it a "project", never "workspace".**

A project has: a set of environments, its own API keys, its own connectors, its own Git connection, its own team membership. Retrieval and the database are **per-environment**, not per-project (so staging and production have separate data).

## Environments

Each project has multiple environments (e.g. `staging`, `production`), plus a separate pool of **dev environments** that back `connic dev` sessions. Environments are fully isolated:

- Separate environment variables / secrets.
- Separate connector configurations.
- Separate database and Retrieval data.
- Separate run history.

Configure under **Project Settings → Git & Environments**. Map a Git branch to each environment for auto-deploy on push. Each environment also has an optional "Test environment" pointer — when set, the deploy gate runs `connic test` against that environment instead of the target, so production deploys can validate with stub credentials.

## Deployment

Two paths:

1. **Git-connected (preferred).** Configure a GitHub/GitLab/Bitbucket repo in **Project Settings → Git & Environments**. Each branch can map to one environment. Pushing runs a three-step pipeline: **Build → Tests → Deploy**. A failing test blocks promotion; git-triggered deploys cannot skip tests.
2. **CLI.** `connic deploy --env=<env-uuid>` from a directory with a valid `.connic`. The `--env` value is an environment UUID, not the human name. `connic deploy` refuses to run if the project has a Git connection — Git is single-source-of-truth in that case. `--skip-tests` is available as a breaking-glass option for non-Git deploys.

GitLab.com accounts connect with OAuth. For Self-Managed GitLab, first add the public HTTPS instance under **Account → Git Accounts** with a personal access token that has the `api` scope, then select that account when connecting the project repository. The GitLab user must be an administrator or have the Maintainer or Owner role on each repository so Connic can manage its webhook.

A deploy bundles the local files (agents, tools, middleware, hooks, schemas, guardrails, tests, requirements.txt) and uploads them to Connic. The new bundle becomes the active version for that environment. Failed builds never replace the live deployment, and previous deployments remain available via an "Activate" action for instant rollback.

Environment variables are injected at deploy time. Changing a variable requires re-deploying for new runs to pick it up.

### PR Testing

Each environment has a **PR Testing** toggle (set in the dashboard under **Project Settings → Git & Environments**). When it's on, every GitHub pull request or GitLab merge request whose target branch matches the environment's branch runs the project's test suite — the same tests `connic test` runs — and the result is posted back as `connic/pr-tests`.

For new environments on git-connected projects with a branch set, PR Testing is on by default. CLI-only projects don't have it.

Each PR only triggers tests in the one environment whose branch matches its target — branch-to-environment is 1:1. Tests run with that environment's variables, secrets, and connections; if the environment has a **Test environment** override set, the tests run there instead (useful for keeping PR traffic out of production).

To require the result before merging:

- **GitHub:** Add `connic/pr-tests` to the branch protection rule. GitHub only lists contexts it has already seen, so run the first PR before selecting it.
- **GitLab:** Connic adds `connic/pr-tests` as an external job in the source branch pipeline. Enable **Settings → Merge requests → Merge checks → Pipelines must succeed**.

PR Testing is supported on GitHub and GitLab.

## Observability

**Dashboard → Logs** and **Dashboard → Runs** show every agent invocation:

- Inputs, outputs, every tool call with args and result.
- Captured `print()` and `logging` calls. Python logger names must start with `tools.`, `middleware.`, `hooks.`, or `guardrails.` — the dashboard surfaces them as "Tool", "Middleware" (tagged `before`/`after`), "Hook", or "Guardrail" entries.
- Token usage, latency, cost per run.
- Reasoning traces (whenever the provider returns them; controlled by `reasoning_effort` on the agent).
- Unhandled exceptions are captured automatically with their tracebacks.

There's a 500-log-lines-per-run cap. Logs view filters include Status, Date Range, Deployment, and Search.

## Retrieval

**Managed semantic retrieval** for each environment. Used via the predefined tools `retrieval_query`, `retrieval_store`, `retrieval_delete`, `retrieval_list_namespaces` (see [predefined-tools.md](predefined-tools.md)).

Dashboard view: **Project → Retrieval** (scoped to the active environment). Inspect entries, edit content, drop namespaces, bulk-import via CSV or file upload.

Namespaces are **dot-separated**, hierarchical, max depth 10 — e.g. `policies.hr.leave`, `products.pricing`. Don't use slashes. Ingestion is asynchronous: `retrieval_store` returns a job ID and entries only become searchable once indexing completes.

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

Serve connector URLs from a subdomain you own while Connic handles TLS. The connector keeps the same ID, signing secret, and configuration; only the hostname changes. Custom domains are available on Pro, Ultimate, and Enterprise.

Add the domain under **Project Settings → Domains**. Apex domains and wildcards are unsupported; use an ASCII subdomain or punycode for an internationalized name. Configure the provided CNAME to `connect.connic.co` and TXT ownership record. DNS propagation can take up to an hour.

Choose the domain in a connector's create or edit dialog. No redeploy or URL rotation is required, and the default `connect.connic.co` URL remains active. Domains are project-scoped and can serve connectors in any project environment. Deleting a domain returns its connectors to the default host; callers using the removed hostname receive 404 responses.

## Team

**Project Settings → Team & Permissions** manages members, reusable permission groups, and the project security policy. Every project has exactly one **Owner** and any number of **Members**:

- The Owner always has full access. Group assignments cannot narrow it. Only the Owner can delete the project or transfer ownership.
- A Member must belong to at least one permission group. Their effective access is the union of every assigned group.
- New projects include editable **Admin** and **User** groups. Admin starts with every permission; User starts with read-and-operate access. These are groups, not fixed roles.
- Custom groups bundle action-level permissions across agents, runs, deployments, connectors, environments, Retrieval, billing, team management, and other areas.
- A group cannot be deleted while a member or pending invite uses it.

The project security policy can require 2FA for every member. Enable 2FA on your own account first; changing the policy requires the **Edit project settings** permission.

API key permissions are separate from team permission groups. Keys can have full access or per-section `read` and `write` access to the REST API.

The immutable audit log under **Project Settings → Audit Log** records the actor, timestamp, action, and before/after values where applicable, with secrets masked. It can be filtered by time, action, resource, or user; retention depends on the plan.

> **Project permissions ≠ end-user authentication.** Permission groups govern who can sign in to the dashboard and manage the project. They do not authenticate the end users whose requests trigger agents. End-user auth (JWTs, OIDC tokens, per-user permissions) belongs in `middleware/<agent>.py::before`, with conditional tools gating actions on the hydrated `context.permissions`. See the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions).

## Usage

**Project → Usage** breaks down LLM token spend and run counts by agent, environment, and period. Budget controls (hard limits, alerts, anomaly detection, scheduled reports) live in the same section.

Architectural recommendations in this skill should be made on the basis of fit, reliability, and maintainability — not cost. Use the dashboard's usage and budget tools to manage spend separately from design decisions.

## REST API

`https://api.connic.co/v1/...`. Auth via API key (create under **Project Settings → CLI / API Keys**). Rate limit: 60 requests/minute per project — 429 responses include `Retry-After`.

The REST API is for **managing and observing a project**: listing runs, reading audit logs, managing deployments, pulling usage and budget data, and managing Retrieval entries, approvals, and judges. Its `POST .../agents/{name}/trigger` endpoint exists only for first-party testing, not for wiring up agent runs.

To run an agent in response to an HTTP request, queue message, email, schedule, or call from a backend, use the matching connector. Connectors provide per-agent URLs, signing secrets, sync/async modes, and replay safety.

For the full endpoint catalogue and per-section permission scopes, fetch `https://connic.co/docs/v1/reference/rest-api`.

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
| Retrieval content | Dashboard or via `retrieval_store` tool |
| Database collections | Dashboard or via `db_*` tools |
| Judges, A/B tests, approvals queue | Dashboard |
| Team members and permission groups | Dashboard |
| Deployment & branch mapping, PR Testing toggle | Dashboard (Git & Environments) |
| Bridge tunnels and custom domains | Dashboard |

Keep this split clear when answering "where do I configure X?" — connectors and secrets are common confusion points and they're dashboard-only.
