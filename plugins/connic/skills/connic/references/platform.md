# Platform (Dashboard) concepts

Use the dashboard at `connic.co` to manage projects, environments, deployments, monitoring, and settings. Agent YAML defines agent behavior; dashboard settings apply to the project or selected environment.

## Project

The top-level scope in Connic. Every dashboard URL is scoped to a project. **Always call it a "project", never "workspace".**

A project has a set of environments, Project credit, API keys, connectors, optional Git connection, and team membership. Retrieval and the database are **per-environment**, not per-project (so staging and production have separate data).

## Models

Deployed Projects support two model paths:

- **Connic-managed:** use an exact `connic/*` ID from the [Connic Model Catalog](https://connic.co/docs/v1/build/connic-models). Models run in the EU and use Project credit at catalog rates; no separate provider credentials are needed.
- **BYOK:** configure credentials under **Project Settings** and use the provider's prefix and model ID. The provider processes and bills the call.

Either can be the primary or fallback model. Never invent a `connic/*` alias or infer an ID from the upstream model name.

For managed inference:

- A `-fast` ID selects a latency-optimized model variant. Its supported reasoning settings, context limit, or input modalities can differ from the standard ID; the catalog lists the exact differences.
- Model-call failures use the agent's configured retry attempts. With `fallback_model`, the primary is tried once and the fallback receives that attempt budget. The run records `context.fallback_model_used` when it switches.
- Every `connic/*` call stays on EU inference capacity and is not used for model training. Catalog warning labels identify non-EU-native upstream providers or a provider's 30-day retention policy; those labels do not change the EU inference location.
- A managed request cancelled by the run timeout or a manual stop settles at zero tokens and releases its reserved Project credit.

## Environments

Every project starts with one default environment and can have more standard environments according to its tier, plus a separate pool of **dev environments** that back `connic dev` sessions. Environments are fully isolated:

- Separate environment variables / secrets.
- Separate connector configurations.
- Separate database and Retrieval data.
- Separate run history.

Configure under **Project Settings → Git & Environments**. Variable keys must be uppercase; sensitive values can be marked **Sensitive**, and the raw editor accepts `KEY=VALUE` lines. Map a Git branch to each environment for auto-deploy on push. Each environment also has an optional "Test environment" pointer — when set, the deploy gate runs `connic test` against that environment instead of the target, so production deploys can validate with stub credentials.

## Deployment

Two paths:

1. **Git-connected (preferred).** Configure a GitHub/GitLab/Bitbucket repo in **Project Settings → Git & Environments**. For a monorepo, set **Repository root directory** to the relative path containing the Connic project. Each branch can map to one environment. Pushing runs a three-step pipeline: **Build → Tests → Deploy**. A failing test blocks promotion; git-triggered deploys cannot skip tests.
2. **CLI.** `connic deploy` from a directory with a valid `.connic` targets the default environment; `connic deploy --env=<env-uuid>` overrides it. The `--env` value is an environment UUID, not the human name. `connic deploy` refuses to run if the project has a Git connection — Git is single-source-of-truth in that case. `--skip-tests` is available as a breaking-glass option for non-Git deploys.

GitLab.com accounts connect with OAuth. For Self-Managed GitLab, first add the public HTTPS instance under **Account → Git Accounts** with a personal access token that has the `api` scope, then select that account when connecting the project repository. The GitLab user must be an administrator or have the Maintainer or Owner role on each repository so Connic can manage its webhook.

A deployment includes agents, tools, middleware, hooks, schemas, guardrails, tests, and `requirements.txt`. A successful deployment becomes active for the environment. Failed builds leave the active deployment intact, and the deployment list provides an **Activate** action for rollback.

Environment variables are injected at deploy time. Changing a variable requires re-deploying for new runs to pick it up.

### PR Testing

Each environment has a **PR Testing** toggle (set in the dashboard under **Project Settings → Git & Environments**). When it's on, every GitHub pull request or GitLab merge request whose target branch matches the environment's branch runs the project's test suite — the same tests `connic test` runs — and the result is posted back as `connic/pr-tests`.

PR Testing defaults to on for Git-connected environments with a branch. CLI-only projects don't have it.

Each PR only triggers tests in the one environment whose branch matches its target — branch-to-environment is 1:1. Tests run with that environment's variables, secrets, and connections; if the environment has a **Test environment** override set, the tests run there instead (useful for keeping PR traffic out of production).

To require the result before merging:

- **GitHub:** Add `connic/pr-tests` to the branch protection rule. GitHub only lists contexts it has already seen, so run the first PR before selecting it.
- **GitLab:** Connic adds `connic/pr-tests` as an external job in the source branch pipeline. Enable **Settings → Merge requests → Merge checks → Pipelines must succeed**.

PR Testing is supported on GitHub and GitLab.

## Observability

**Project → Agent Runs** lists executions; **Project → Logs** aggregates captured custom-code log lines across runs. Run details connect the two:

- Inputs, outputs, every tool call with args and result.
- Captured `print()` and `logging` calls. Python logger names must start with `tools.`, `middleware.`, `hooks.`, or `guardrails.` — the dashboard surfaces them as "Tool", "Middleware" (tagged `before`/`after`), "Hook", or "Guardrail" entries.
- Token usage, latency, cost per run.
- Reasoning traces (whenever the provider returns them; controlled by `reasoning_effort` on the agent).
- Unhandled exceptions are captured automatically with their tracebacks.

There's a 500-log-lines-per-run cap. Agent Runs filters include Status, Date Range, Deployment, and Search; Logs can be filtered and searched across captured lines.

A run detail includes the raw/formatted input, output, final context, error, parent/connector metadata, token breakdown, duration, and a hierarchical trace. Span types cover logical LLM steps, physical provider calls and retries, local tools, MCP tools, middleware, sequential steps, and the root run/loop. **Run Again** repeats the same agent and input; queued or running executions can be cancelled.

The **Observability** tab also supports multiple drag-and-drop dashboards. Widgets include stat cards, area and bar charts, recent-log lists, and shared text/select inputs; select options can come from Database collections. Dashboards have global date ranges, per-widget agent/connector filters, shared variables, and optional 10-second refresh. An agent's detail page has its own run statistics, status breakdown, configuration, filtered history, and manual trigger.

Run filters and dashboard widgets use safe expressions over `context.*`, `input.*`, and `output.*`. Log and audit drains use the same grammar with `entry.*` bound to the candidate entry; unknown roots, calls, arithmetic, assignments, and private attributes are rejected, and expressions are capped at 2,000 characters.

Token Usage reports `connic/*` costs in EUR. BYOK usage retains the USD or EUR currency configured for that model; different currencies are displayed separately and never converted or combined. Project Billing records the authoritative Project-credit debit for managed inference, while the BYOK provider's bill remains authoritative for its calls.

## Retrieval

**Managed semantic retrieval** for each environment. Used via the predefined tools `retrieval_query`, `retrieval_store`, `retrieval_delete`, `retrieval_list_namespaces` (see [predefined-tools.md](predefined-tools.md)).

Dashboard view: **Project → Retrieval** (scoped to the active environment). Upload content, inspect and filter indexed entries, run semantic Search, delete entries, and monitor ingestion jobs. Content updates use another upload/store with the same namespace-local entry ID.

Use **dot-separated** namespace names for hierarchy, with a maximum depth of 10 — e.g. `policies.hr.leave`, `products.pricing`. Searching a namespace includes all descendants. Ingestion is asynchronous: `retrieval_store` returns a job ID and entries only become searchable once indexing completes.

Retrieval has no fixed entry-count cap. Stored content contributes to project storage usage and Project credit. Notion, Confluence Cloud, Superhuman Docs (Coda), and website sources can sync on a schedule or on demand; each successfully ingested source item uses Project credit at the retrieval-source rate.

## Database

**Per-environment** schemaless document store. Used via `db_find` (including distinct-value queries), `db_insert`, `db_update`, `db_upsert`, `db_delete`, `db_count`, and `db_list_collections` (see [predefined-tools.md](predefined-tools.md)).

Dashboard view: **Project → Database**. Browse documents with filter/sort controls, generate filters and sorting from plain language, insert documents manually, inspect inferred schemas and fill rates, and create or delete collections.

Production data is never visible from staging.

## Judges

Automated evaluators score completed runs for correctness, helpfulness, safety, or custom criteria. Configure them in **Project → Judges**. A judge takes a completed run as input and emits a score and reason.

- Judges are **per-agent only** — set one up against the specific agent you want graded.
- Judges can use an exact `connic/*` model or any BYOK provider configured for the Project.
- Triggers: automatic on every run, automatic on a sample (configurable rate), or manual.
- Filter expressions over `input.*`, `output.*`, and `context.*` let you grade only matching runs. The judge receives the run's status, input, output, error, public context, token usage, and up to 100 recent trace spans. It receives the agent's system prompt only when **Include agent system prompt** is enabled. Its overall score is the sum of independently scored criteria.
- Each completed judge evaluation consumes one additional Project-credit run unit; a failed evaluation consumes zero judge run units.
- Score alerts use a configurable rolling window (last 10 completed evaluations by default) and fire only when the average crosses from at/above the threshold to below it. They can fire again after recovery and a later drop.

## Approvals

Human-in-the-loop gating for specific tool calls. Configured in the agent YAML (`approval:` block — see [agent-yaml.md](agent-yaml.md)). When an agent hits a gated tool, the run pauses and appears in **Project → Approvals**. A reviewer approves or rejects; the run resumes (or fails / skips the tool depending on `on_rejection`). Conditions on `param.*` and `context.*` let you gate only some calls; an evaluation error requires approval. Project members receive email and in-app notifications according to their notification preferences. Approval settings can also send a signed webhook. Multiple gated calls create separate pause/resume cycles; each resume loads the full conversation history without re-executing tools whose approvals are already recorded. Decisions are submitted in Connic or through the REST API, and are recorded in both the audit log and run trace — notification webhooks do not decide approvals.

## Bridge

A tunnel from Connic's cloud to a private network (your VPC, on-prem services). Provision a Bridge under **Project Settings → Bridge**, copy its token when it is shown once, then install the Bridge agent on a machine inside your network. The agent requires `BRIDGE_TOKEN` and a comma-separated `ALLOWED_HOSTS`; optional settings are `RELAY_URL` (default `wss://relay.connic.co`) and `LOG_LEVEL` (default `INFO`). The Bridge ID can then be referenced from four kinds of consumer:

1. **Connectors** — pick the Bridge in the connector config dropdown. Bridge-capable types are Kafka and SQS (both directions), Postgres, email (both directions), S3 file downloads, and outbound webhook callbacks.
2. **Custom LLM providers** — pick the Bridge in the provider config.
3. **Custom tools / middleware / hooks / guardrails** — reach private endpoints via the magic hostname `<target>.cnc-bridge-<bridge_id>` from inside your Python code.
4. **MCP servers** — set the `bridge:` field on the `mcp_servers` entry.

```yaml
mcp_servers:
  - name: internal
    url: http://mcp.internal:8080/mcp
    bridge: ${INTERNAL_BRIDGE_ID}
```

```python
# From a custom tool reaching a private internal API
import os
import httpx
url = f"https://api.internal.cnc-bridge-{os.environ['INTERNAL_BRIDGE_ID']}/users"
r = await httpx.AsyncClient().get(url)
```

For protocols that discover endpoints at runtime, such as Redis Sentinel, configure automatic destination routes under **Project Settings → Bridge** or with `GET/PUT /v1/projects/{project_id}/bridges/{bridge_id}/routes`. The user or credential needs the `bridges.update` project permission:

```json
{
  "routes": [
    {"match_type": "exact", "target": "redis-sentinel.internal", "port": 26379},
    {"match_type": "regex", "target": "^redis-[a-z0-9-]+\\.internal$", "port": 6379}
  ]
}
```

- A bridge supports up to 32 routes, with 256 routes total across a project. Each route matches one hostname or IP and one TCP port.
- Regex routes use the safe anchored `^...$` subset. They cannot contain groups, lookarounds, backreferences, alternation, braces, unescaped dots, or broad `.*` patterns, and may contain at most one quantified character class such as `[a-z0-9-]+`.
- Resolution order is explicit magic hostname, exact route, then regex route. Matches across distinct bridge IDs fail closed.
- A route selects a tunnel but does not grant access. The bridge agent's exact `ALLOWED_HOSTS` must include the Sentinel and every possible master, such as `redis-sentinel.internal:26379,redis-1.internal:6379,redis-2.internal:6379`.
- Automatic routes preserve the original hostname for TLS verification. With an explicit magic hostname, set SNI / `server_hostname` to the real target when the client verifies certificates.
- Use `<target>.cnc-bridge-<bridge_id>` with custom DNS resolvers and connections opened from background threads.

## Domains

Serve connector URLs from a subdomain you own while Connic handles TLS. The connector keeps the same ID, signing secret, and configuration; only the hostname changes. Custom domains are available on Pro and Enterprise.

Add the domain under **Project Settings → Domains**. Apex domains and wildcards are unsupported; use an ASCII subdomain or punycode for an internationalized name. Configure the provided CNAME to `connect.connic.co` and TXT ownership record. DNS propagation can take up to an hour.

Choose the domain in a connector's create or edit dialog. No redeploy or URL rotation is required, and the default `connect.connic.co` URL remains active. Domains are project-scoped and can serve connectors in any project environment. Deleting a domain returns its connectors to the default host; callers using the removed hostname receive 404 responses. A downgrade below Pro also makes custom hosts return 404 while default URLs keep working. Domain changes require the corresponding project action permission and are audit-logged.

## Team

**Project Settings → Team & Permissions** manages members, reusable permission groups, and the project security policy. Every project has exactly one **Owner** and any number of **Members**:

- The Owner always has full access. Group assignments cannot narrow it. Only the Owner can delete the project or transfer ownership.
- A Member must belong to at least one permission group. Their effective access is the union of every assigned group.
- Projects include editable **Admin** and **User** groups. Admin starts with every permission; User starts with read-and-operate access. These are groups, not fixed roles.
- Custom groups bundle action-level permissions across agents, runs, deployments, connectors, environments, Retrieval, billing, team management, and other areas.
- A group cannot be deleted while a member or pending invite uses it.

The project security policy can require 2FA for every member. Enable 2FA on your own account first; changing the policy requires the **Edit project settings** permission.

Team permission groups are the source of truth for project access. API keys and MCP authorizations can use all supported permissions available to their authorizing user or a selected subset, but they can never exceed that user's current access. Role, group, and membership changes take effect on subsequent credential requests.

The immutable audit log under **Project Settings → Audit Log** records the actor, timestamp, action, and resource state where applicable, with secrets masked. It can be filtered by time, action, resource, or user; retention depends on the plan.

> **Project permissions ≠ end-user authentication.** Permission groups govern who can sign in to the dashboard and manage the project. They do not authenticate the end users whose requests trigger agents. End-user auth (JWTs, OIDC tokens, per-user permissions) belongs in `middleware/<agent>.py::before`, with conditional tools gating actions on the hydrated `context.permissions`. See the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions).

## Usage

**Project → Token Usage** breaks down model-token analytics and run counts by agent, environment, and period. Managed `connic/*` token cost is shown in EUR; BYOK uses its configured USD or EUR currency, and currencies are never converted or combined. These analytics power model-cost limits, alerts, anomaly detection, and scheduled reports and remain separate from the Project-credit ledger.

Token-cost **alerts** notify without stopping runs; **limits** hold affected new runs. Either can be global, environment-scoped, or agent-scoped and reset daily at midnight UTC or monthly on the first. Held runs are released when the limit is disabled/deleted, raised above current spend, or its period resets. Optional anomaly detection compares each completed run with that agent's 30-day rolling average (default threshold `3x`, minimum five completed runs). Weekly reports arrive Monday at 09:00 UTC for the prior seven days; monthly reports arrive on the first at 09:00 UTC for the prior month.

**Project → Billing** shows the Project credit balance and consumption across runs, compute, storage, retrieval, and `connic/*` model tokens. Basic includes a one-time plan credit; Developer and Pro include monthly credit. Standard Projects can buy credit or configure capped auto-refill. Prepaid usage stops when credit is insufficient. Approved Enterprise Projects can use monthly postpaid billing. Stripe handles payments, receipts, and invoices.

Architectural recommendations should start with fit, reliability, and maintainability, then compare cost and latency among choices that meet those requirements. Use the dashboard's usage and budget tools to manage spend and make the tradeoff explicit.

## REST API

`https://api.connic.co/v1/...`. Create a project-scoped API key under **Project Settings → API Keys & MCP Auth** and send `Authorization: Bearer cnc_...`; the secret is shown only once. Keys default to **All available**, which follows every REST API permission the key owner has. Choose **Custom** to grant an explicit subset of those same project permissions, and edit that subset without rotating the secret. A key never exceeds its owner's live project access. All keys for a project share one 60-requests/minute bucket; 429 responses include `Retry-After`. Standard errors use a JSON `{ "detail": "..." }` body.

The REST API is for **managing and observing a project**: listing runs, reading audit logs, managing deployments, pulling usage and budget data, and managing Retrieval entries, approvals, and judges. Use connectors to start runs from external events.

To run an agent in response to an HTTP request, queue message, email, schedule, or call from a backend, use the matching connector. Connectors provide transport-specific endpoints, authentication, sync/async behavior, and delivery semantics. Do not assume generic replay protection: apply idempotency in project code where the source can redeliver.

For the full endpoint catalogue and its canonical project-permission requirements, fetch `https://connic.co/docs/v1/reference/rest-api`.

Use the interface that matches the job:

- **Connic MCP** for conversational inspection and bounded project operations from an OAuth-capable AI client. See [platform-mcp.md](platform-mcp.md).
- **REST API** for deterministic service integrations and repeatable automation.
- **CLI** for local authoring, validation, tests, and deployment workflows.

## What lives where: dashboard vs. on-disk

| Concern | Lives in… |
| --- | --- |
| Agent definition (prompt, tools, model, etc.) | On-disk YAML (`agents/`) |
| Tool implementations | On-disk Python (`tools/`) |
| Middleware / hooks / guardrails | On-disk Python |
| JSON output schemas | On-disk (`schemas/`) |
| Declarative tests | On-disk (`tests/`) |
| Environment variables / secrets | Dashboard (per environment) |
| BYOK model-provider credentials | Dashboard (Project Settings) |
| Project credit, top-ups, auto-refill, and Stripe documents | Dashboard (Project → Billing) |
| Connector configuration | Dashboard (linked to agents) |
| API spec imports | Dashboard |
| Retrieval content | Dashboard or via `retrieval_store` tool |
| Database collections | Dashboard or via `db_*` tools |
| Judges, A/B tests, approvals queue | Dashboard |
| Team members and permission groups | Dashboard |
| Deployment & branch mapping, PR Testing toggle | Dashboard (Git & Environments) |
| Bridge tunnels and custom domains | Dashboard |

Keep this split clear when answering "where do I configure X?" — connectors and secrets are common confusion points and they're dashboard-only.
