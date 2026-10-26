# Platform (Dashboard) concepts

Use the dashboard at `connic.co` to manage projects, environments, deployments, monitoring, and settings. Agent YAML defines agent behavior; dashboard settings apply to the project or selected environment.

## Project

The top-level scope in connic. Every dashboard URL is scoped to a project. **Always call it a "project", never "workspace".**

A project has a set of environments, Project credit, API keys, connectors, optional Git connection, and team membership. Retrieval and the database are **per-environment**, not per-project (so staging and production have separate data).

## Models

Deployed Projects support two model paths:

- **connic-managed:** use an exact `connic/*` ID from the [connic model catalog](https://connic.co/docs/v1/build/connic-models). Models run in the EU and use Project credit at catalog rates; no separate provider credentials are needed.
- **BYOK:** configure credentials under **Project Settings** and use the provider's prefix and model ID. The provider processes and bills the call.

Either can be the primary or fallback model. Never invent a `connic/*` alias or infer an ID from the upstream model name.

For managed inference:

- A `-fast` ID selects a latency-optimized model variant. Its supported reasoning settings, context limit, or input modalities can differ from the standard ID; the catalog lists the exact differences.
- Model-call failures use the agent's configured retry attempts, with a default 120-second cap per request, limited by remaining run time. Without a fallback, the primary receives the attempt budget; with `fallback_model`, the primary is tried once and the fallback receives that budget. Context compression allows one separate same-model repair request before fallback. Cancellation, run expiry, or emitted streaming output stops further attempts. See [LLM retries and timeouts](agent-yaml.md#llm-retries-and-timeouts). The run records `context.fallback_model_used` when it switches.
- Every `connic/*` call stays on EU inference capacity and is not used for model training. Catalog warning labels identify non-EU-native upstream providers or a provider's 30-day retention policy; those labels do not change the EU inference location. Models without the retention label run only on providers with a zero-data-retention policy.
- A managed request cancelled by the run timeout or a manual stop settles at zero tokens and releases its reserved Project credit.
- Models with the Files capability read PDFs natively, including scanned pages; the catalog tooltip lists each model's accepted file types. GPT-5.4 and GPT-5.6 also accept Word, Excel, PowerPoint, RTF, OpenDocument, and EPUB. Text files (txt, csv, tsv, md, html, json, jsonl, xml) reach every model as text. Any other file in the latest message fails the request with `unsupported_file_type`. connic replaces unsupported attachments in earlier messages with text notes so later requests can continue. A request can be up to 48 MiB after base64 encoding, supported files stay in the conversation for later requests in the run, and `fallback_model` should accept the same file types: a fallback that rejects a file in the latest message fails with `unsupported_file_type`, and the deployment log warns about such a pair. Token cost per page differs by model, and scanned pages can cost more than text pages. See [file input](https://connic.co/docs/v1/build/connic-models#file-input).

## Environments

Every project starts with one default environment and can have more standard environments according to its tier, plus a separate pool of **dev environments** that back `connic dev` sessions. Environments are fully isolated:

- Separate environment variables / secrets.
- Separate connector configurations.
- Separate database and Retrieval data.
- Separate run history.

Configure under **Project Settings → Git & Environments**. Variable keys must be uppercase; sensitive values can be marked **Sensitive**, and the raw editor accepts `KEY=VALUE` lines. Map a Git branch to each environment for auto-deploy on push. Each environment also has an optional "Test environment" pointer — when set, the deploy gate runs `connic test` against that environment instead of the target, so production deploys can validate with stub credentials.

The environment drawer also has **On-demand deployments** and **Copy from this env** group selectors. Both default to **All groups** for existing and new environments until a restriction is saved. On-demand deployment groups restrict manual deployments in addition to the existing project and API-key permissions. Owners retain access; when specific groups are selected, administrators must also belong to one. For either selector, **All groups** adds no group restriction, selected groups limit access to those groups, and an empty selection permits owners only. The same group restrictions apply to manual test uploads, dev-session updates, activation, redeployment, and overriding failed deployment tests. Automatic branch deployments and PR workflows are unaffected. **Copy from this env** controls copying from a normal source environment to other normal or dev environments in the same project. The Retrieval, Database, and API Tools copy drawers select an allowed source and items in a tree, then copy into the current environment. On Variables, select rows in an allowed normal source environment and choose **Copy to other envs** to copy to one or more other normal or dev environments in the same project. Database supports a JSON document filter per selected collection; Retrieval supports content type, entry ID substring, and metadata filters. Existing items are skipped by default; choose **Replace** to overwrite matching items. Existing resource permissions still apply, including update permission for replacing variables or API tools. Sensitive variable values and API authentication credentials stay on the server. API tools retain customized names, descriptions, and enabled selections.

Each environment uses its plan's log-retention period by default and can set a shorter period from 1 hour up to the plan limit. Log retention deletes runs automatically. Sensitive-data redaction replaces values at configured, case-sensitive JSON paths in newly recorded structured inputs, outputs, run context, persistent session state, tool calls, logs, and traces. Input paths start at the connector payload; the saved context keeps a copy under `payload`, so a leading `*.` (e.g. `*.user_token`) covers the input, `context["payload"]`, and a custom `context["user_token"]`. Agents, tools, and callers keep the original values. Existing history, plain text, invalid JSON, and JSON embedded in longer text are not redacted.

## Deployment

Two paths:

1. **Git-connected (preferred).** Configure a GitHub/GitLab/Bitbucket repo in **Project Settings → Git & Environments**. For a monorepo, set **Repository root directory** to the relative path containing the connic project. Each branch can map to one environment. Pushing runs a three-step pipeline: **Build → Tests → Deploy**, and a failing test blocks promotion. The Deployments page can also start the mapped branch manually with **Deploy now** or **Deploy & skip tests**. Automatic deployments triggered by a push always run the gate.
2. **CLI.** `connic deploy` from a directory with a valid `.connic` lists environments by name and asks for confirmation before uploading. `connic deploy --list` lists targets without deploying. `--env <name-or-id>` selects a target directly; use `--env staging --yes` for CI. `connic deploy` refuses to run if the project has a Git connection — Git is single-source-of-truth in that case. `--skip-tests` is available as a breaking-glass option for non-Git deploys.

GitLab.com accounts connect with OAuth. For Self-Managed GitLab, first add the public HTTPS instance under **Account → Git Accounts** with a personal access token that has the `api` scope, then select that account when connecting the project repository. The GitLab user must be an administrator or have the Maintainer or Owner role on each repository so connic can manage its webhook.

A deployment includes agents, tools, middleware, hooks, schemas, guardrails, tests, connectors (`connectors/*.yaml`), and `requirements.txt`. A successful deployment becomes active for the environment. Failed builds leave the active deployment intact, and the deployment list provides an **Activate** action for rollback. Activation also applies the deployment's code connectors, so a rollback restores them; see [Connectors in code](connectors.md#connectors-in-code). A run uses the deployment that is active when it starts, so runs still queued at activation (including rollback activation) start on the newly active deployment.

connic reviews the submitted deployment files before the build. A deployment stopped by this review is marked **Blocked**. connic does not build, test, or activate it, and the deployment details show instructions for contacting support.

Environment variables are injected at deploy time. Changing a variable requires re-deploying for new runs to pick it up.

### PR Testing

Each environment has a **PR Testing** toggle (set in the dashboard under **Project Settings → Git & Environments**). When PR Testing is enabled, connic runs the project's test suite for GitHub pull requests and GitLab merge requests whose source branch is in the connected repository and whose target branch matches the environment's branch. connic runs the same tests as `connic test` and posts the result as `connic/pr-tests`. Fork-based pull requests and merge requests do not trigger PR Testing.

PR Testing defaults to on for Git-connected environments with a branch. CLI-only projects don't have it.

Each PR only triggers tests in the one environment whose branch matches its target — branch-to-environment is 1:1. Tests run with that environment's variables, secrets, and connections; if the environment has a **Test environment** override set, the tests run there instead (useful for keeping PR traffic out of production).

To require the result before merging:

- **GitHub:** Add `connic/pr-tests` to the branch protection rule. GitHub only lists contexts it has already seen, so run the first PR before selecting it.
- **GitLab:** connic adds `connic/pr-tests` as an external job in the source branch pipeline. Enable **Settings → Merge requests → Merge checks → Pipelines must succeed**.

PR Testing is supported on GitHub and GitLab.

## Observability

**Project → Agent Runs** lists executions; **Project → Logs** aggregates captured custom-code log lines across runs. Run details connect the two:

- Inputs, outputs, every tool call with args and result.
- Captured `print()` and `logging` calls. Python logger names must start with `tools.`, `middleware.`, `hooks.`, or `guardrails.` — the dashboard surfaces them as "Tool", "Middleware" (tagged `before`/`after`/`cleanup`), "Hook", or "Guardrail" entries.
- Token usage, latency, cost per run.
- Reasoning traces (whenever the provider returns them; controlled by `reasoning_effort` on the agent).
- Unhandled exceptions are captured automatically with their tracebacks.

Open **Playground** from the project navigation to run an active agent. Text, tool, and sequential agents accept Text or JSON input, up to 10 file attachments with a combined size of 10 MiB, and scheduled runs. Voice agents start a live microphone session. Choose the base agent or a compatible deployed test variant; Playground uses the selected environment's tools and connections and shows output, traces, logs, and recent manual runs in place. Using it requires `agents.view`, `runs.view`, and `runs.trigger`; viewing its recent-run list also requires `stats.view`.

A run stopped by a safety review is marked `blocked`, does not continue processing, and shows instructions for contacting support in the run details.

There's a 500-log-lines-per-run cap. Agent Runs filters include Status, Date Range, Deployment, and Search; Logs can be filtered and searched across captured lines.

A run detail includes the raw/formatted input, output, final context, error, parent/connector metadata, token breakdown, duration, and a hierarchical trace. Span types cover logical LLM steps, physical provider calls and retries, local tools, MCP tools, middleware, triggered-agent links, and the root run/loop. Each sequential step is persisted as a separate child run; follow its Triggered Agent span to inspect it. The completed-run header excludes recorded approval waits; run lists and aggregate metrics use backend active duration, which also excludes sequential child-run waits. Cancelling a sequential parent also cancels its unfinished child chain. **Run Again** repeats the same agent and input; queued or running executions can be cancelled.

Completed runs can be deleted individually or in bulk after their background processing has finished. The user needs the **Delete runs** project permission. Deletion is permanent. Log retention deletes runs automatically after the configured period.

The **Observability** tab also supports multiple drag-and-drop dashboards. Widgets include stat cards, area and bar charts, recent-log lists, and shared text/select inputs; select options can come from Database collections. Dashboards have global date ranges, per-widget agent/connector filters, shared variables, and optional 10-second refresh. For cost per customer or use case, have middleware set a context key such as `context["customer_id"]` and add a bar chart grouped **By Context** on that key with the Token Cost or Token Usage metric (top 20 values per environment and date range). Widget filters accept input-variable placeholders as quoted comparison operands, e.g. `context.customer_id == '{customer}'`; an empty input drops that condition. Context keys may be nested with dots (`customer.id`). An agent's detail page has its own run statistics, status breakdown, configuration, filtered history, and manual trigger.

Enterprise projects forward data to external systems through **channels** (**Project Settings → Notifications**, `channels.manage`). Channel types: **Webhook** (JSON array POST, optionally signed with `X-Connic-Timestamp` and `X-Connic-Signature`, the hex HMAC-SHA256 of `{timestamp}.{raw body}`), **Datadog** (Logs intake with API key and site; `service` is the agent name, `env` tag the environment name, status from `level`, all other fields are attributes), and **OpenTelemetry** (OTLP/HTTP JSON to an endpoint base URL: spans to `/v1/traces`, every other entry to `/v1/logs`, with optional `key=value` headers; `service.name` is the agent name and log records use the run ID as trace ID). **Slack** and **Microsoft Teams** are listed as "On request" and enabled when a customer asks connic support. Webhook and OpenTelemetry channels can deliver through a connic bridge to private endpoints, including plain HTTP such as an in-cluster Datadog Agent OTLP receiver; the bridge's `ALLOWED_HOSTS`, if set, must include the endpoint. A channel can subscribe to notification events such as `agent_runs_failing` (3 of an agent's last 5 finished runs failed), `build_failed`, `judge_score_low`, and `budget_alert_triggered`, receive approval requests, and serve drains.

Drains (https://connic.co/docs/v1/platform/observability/drains) forward new entries to one channel. **Run drains** (**Agent Runs → More actions → Run Drains**) forward everything by default, untruncated, and each part can be switched off: a run event per finished run, including deploy-gate test runs, about a minute after it ends (`run_status`, `level`, `agent_name`, `environment_name`, `trigger_source`, `duration_ms`, `queue_wait_ms`, `token_usage`, `error`, public `context`, and the run's full `input` and `output`, parsed when JSON); every span of each run with the run ID as trace ID, parent links, timing, status, attributes, and full input, output, and reasoning (OTLP traces on OpenTelemetry channels with `connic.input`, `connic.output`, and `connic.thoughts`; one `type: "span"` entry per span on Webhook and Datadog channels; Datadog APM needs an OpenTelemetry channel to the Datadog Agent); and an `environment_health` snapshot at the start of every minute per environment plus per agent with active runs (`queued_runs`, `running_runs`, `awaiting_approval_runs`, `oldest_queued_seconds`, `oldest_running_seconds`). Run events and traces filter by environment, run status, trigger source, and an `entry.*` expression that can read `entry.context`, `entry.input`, and `entry.output` even when they are not forwarded. Health snapshots use only the environment selection and the health toggle. **Log drains** (**Logs → Log Drains**) batch captured log lines every 15 seconds, up to 500 per request, each with its run's `trigger_source`, filtered by environment, level, source, trigger source, and expression. **Audit drains** (**Project Settings → Audit Log**) forward audit entries about every 60 seconds; guardrail blocks arrive as `guardrail.block`. Drains start at creation without backfill, retry failed event, trace, log, and audit deliveries on the next cycle, can deliver an entry twice (deduplicate on `id`, the run ID for run events), and pause when their channel is disabled. Failed health snapshots are replaced by fresh snapshots the next minute. A failed run never reaches `after()` middleware, so use a run drain to export failures.

Run filters and dashboard widgets use safe expressions over `context.*`, `input.*`, and `output.*`. Drains use the same grammar with `entry.*` bound to the candidate entry; unknown roots, calls, arithmetic, assignments, and private attributes are rejected, and expressions are capped at 2,000 characters.

Managed `connic/*` token costs use EUR; BYOK costs use the USD or EUR currency configured for that model. The project's **Cost Display** setting in **Token Pricing** can convert reported costs to EUR or USD using the European Central Bank daily reference rate for each call date. Converted totals are estimates and can differ from provider bills. Project Billing records the authoritative Project-credit debit for managed inference, while the BYOK provider's bill remains authoritative for its calls.

## Retrieval

**Managed semantic retrieval** for each environment. Used via the predefined tools `retrieval_query`, `retrieval_store`, `retrieval_delete`, `retrieval_list_namespaces` (see [predefined-tools.md](predefined-tools.md)).

Dashboard view: **Project → Retrieval** (scoped to the active environment). Upload content, inspect and filter indexed entries, run semantic Search, delete entries, and monitor ingestion jobs. Content updates use another upload/store with the same namespace-local entry ID.

Use **dot-separated** namespace names for hierarchy, with a maximum depth of 10 — e.g. `policies.hr.leave`, `products.pricing`. Searching a namespace includes all descendants. Ingestion is asynchronous: `retrieval_store` returns a job ID and entries only become searchable once indexing completes.

Text uploads (`POST /v1/projects/{project_id}/retrieval/text`, `retrieval_store`) accept up to 10,000,000 characters, `upload_retrieval_text` up to 250,000. Upload longer documents as a file: `POST /v1/projects/{project_id}/retrieval/file` takes up to 100 MiB and 25,000,000 characters of text, `upload_retrieval_file` up to 512 KiB. Both upload types store metadata on every chunk, so `metadata_filter` can match it. Text uploads take a `metadata` object, and so does `upload_retrieval_file`. REST file uploads take a `metadata` multipart form field holding a JSON object. Files always get `file_size` and `original_filename` from the upload, which replace user keys of the same name.

File uploads accept PDF, PNG, JPG/JPEG, GIF, WebP, Word (DOCX, DOC), Excel (XLSX, XLS), PowerPoint (PPTX), OpenDocument (ODT, ODS, ODP), RTF, HTML, email (EML, MSG), and text files (TXT, Markdown, CSV, JSON, JSONL, YAML, LOG), so upload Office files and mails as they are instead of converting them to text. The extension must match the content; password-protected files are rejected. Entries get the content type `text`, `pdf`, `image`, `document`, `spreadsheet`, `presentation`, or `email`. Spreadsheet rows are indexed as `Header: value` pairs, up to 10,000 rows and 200 columns per sheet, and every chunk starts with the sheet name, also stored as `sheet_name` metadata. Presentation chunks return the slide number as `page_number`. Email chunks start with subject, sender, and date, also stored as `email_subject`, `email_from`, `email_to`, `email_cc`, and `email_date` metadata. Mail attachments are listed by name but not indexed; upload them as separate files.

Retrieval has no fixed entry-count cap. Stored content contributes to project storage usage and Project credit. Notion, Confluence Cloud, Superhuman Docs (Coda), Zendesk Help Center, SharePoint and OneDrive, and website sources can sync on a schedule or on demand; each successfully ingested source item uses Project credit at the retrieval-source rate.

A SharePoint and OneDrive source (`type: sharepoint`) keeps document libraries searchable without custom sync code against the Retrieval REST API. It signs in with the customer's Microsoft Entra ID app registration: `tenant_id`, `client_id`, and `client_secret`, plus `site_urls` (one per line) and/or `onedrive_users` (sign-in emails), optionally `library_names`, `folder_paths` (one per line from the top of each library or OneDrive, such as `Sales/Offers`; only files in these folders and their subfolders sync, and a listed folder missing from every selected library fails the sync), `modified_since` (`YYYY-MM-DD`), and `include_images` (default `false`). The app needs a Microsoft Graph application permission with admin consent: `Sites.Selected` with a read grant on each site, or `Files.Read.All` for OneDrive and every site. After the first sync, a run only downloads files that changed; a large first sync continues over several runs automatically. Files Retrieval cannot ingest are skipped and counted in the run's `file_issues`.

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

Human-in-the-loop gating for specific tool calls. Configured in the agent YAML (`approval:` block — see [agent-yaml.md](agent-yaml.md)). When an agent hits a gated tool, the run pauses and appears in **Project → Approvals**. A reviewer approves or rejects; the run resumes (or fails / skips the tool depending on `on_rejection`). On LLM agents, `approval.inputs` creates tools that require a human text response and return it to the model; no Python function is needed. Conditions on `param.*` and `context.*` let you gate only some calls; an evaluation error requires approval. Project members with `approvals.view` receive email and in-app notifications according to their notification preferences. Approval settings can also send a signed webhook. Multiple gated calls create separate pause/resume cycles; each resume loads the full conversation history without re-executing tools whose approvals are already recorded. Decisions are submitted in connic or through the REST API, and are recorded in both the audit log and run trace — notification webhooks do not decide approvals.

## Bridge

A tunnel from connic's cloud to a private network (your VPC, on-prem services). Provision a bridge under **Project Settings → Bridge**, copy its token when it is shown once, then install the bridge agent on a machine inside your network. The agent requires `BRIDGE_TOKEN`. `ALLOWED_HOSTS` is an optional comma-separated exact `host:port` allowlist; when it is unset or empty, every target reachable from the bridge's network is allowed. Other optional settings are `RELAY_URL` (default `wss://relay.connic.co`) and `LOG_LEVEL` (default `INFO`). The bridge ID can then be referenced from four kinds of consumer:

1. **Connectors** — Kafka and SQS (both directions), Postgres, email (both directions), and S3 file downloads from a custom `endpoint_url` use the bridge of their **Connection**: pick it in the Connection drawer, or send the Connection's top-level `bridge` (name or ID) through REST/MCP. Every connector on that Connection routes through it; use a different Connection per environment for different bridges. Outbound webhook callbacks pick the bridge on the connector (`bridge` in `connectors/*.yaml`, REST, and MCP).
2. **Custom LLM providers** — pick the bridge in the provider config.
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
url = f"http://api.internal.cnc-bridge-{os.environ['INTERNAL_BRIDGE_ID']}/users"
async with httpx.AsyncClient(timeout=30) as client:
    r = await client.get(url)
```

- Sync and async clients both work: httpx (`Client` and `AsyncClient`), requests, aiohttp, anyio, and asyncio streams. Clients with their own DNS resolver bypass the bridge; for aiohttp with `aiodns` installed, pass `aiohttp.TCPConnector(resolver=aiohttp.ThreadedResolver())`. httpx gives up after 5 seconds by default, so set `timeout=` for slow internal calls.
- A service that only listens on the bridge machine (for example `127.0.0.1:8080`) is reached with target `localhost`: `http://localhost.cnc-bridge-<bridge_id>:8080`. `ALLOWED_HOSTS` matches the exact `host:port` of the request, so this call needs `localhost:8080`; `127.0.0.1:8080` does not match. Inside Docker, `localhost` is the bridge container, and Docker on a Mac cannot reach the Mac's localhost, so run the pip agent (`pip install connic-bridge`, Python 3.10+) directly on the Mac.
- The agent reconnects after network drops but exits when the token is rejected or the project has no active deployment. Run the pip agent under a service manager: a macOS LaunchAgent with `KeepAlive` set to true, or a systemd unit with `Restart=always`. With Docker use `--restart unless-stopped` or `restart: always`. Full launchd and systemd files are in the Bridge docs (`/docs/v1/platform/bridge#run-as-service`).
- From connic-bridge 0.1.7, responses and uploads stream with per-channel backpressure, so large files arrive intact when one side reads slowly; older agents cut such transfers. When a tunnel cannot open, HTTP/1.x clients get `502` with the body `connic bridge: <reason>` (`Bridge is not connected`, or `Bridge could not open host:port` when the bridge rejects or cannot reach the target); other protocols see the connection close.
- Deleting a bridge is refused (409, naming them) while a connector, Connection, notification channel, or OAuth app uses it; remove the bridge from those first.

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

Serve connector URLs from a subdomain you own while connic handles TLS. The connector keeps the same ID, signing secret, and configuration; only the hostname changes. Custom domains are available on pro and enterprise.

Add the domain under **Project Settings → Domains**. Apex domains and wildcards are unsupported; use an ASCII subdomain or punycode for an internationalized name. Configure the provided CNAME to `connect.connic.co` and TXT ownership record. DNS propagation can take up to an hour.

Choose the domain in a connector's create or edit dialog. No redeploy or URL rotation is required, and the default `connect.connic.co` URL remains active. Domains are project-scoped and can serve connectors in any project environment. Deleting a domain returns its connectors to the default host; callers using the removed hostname receive 404 responses. A downgrade below pro also makes custom hosts return 404 while default URLs keep working. Domain changes require the corresponding project action permission and are audit-logged.

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

**Project → Token Usage** breaks down model-token analytics and run counts by agent, model, or run context value (**By Context**, e.g. `customer_id`; runs without the key show as "(not set)"), per environment and month. The REST and MCP equivalents are `GET /v1/projects/{project_id}/budgets/daily-cost` and `/budgets/agent-ranking` (`get_budget_daily_cost`, `get_budget_agent_ranking`) with `group_by=agent`, `model`, or `context.<key>`; REST context rows carry `context_value`, while MCP context rows use `name` (`null` when unset). Use these for per-customer cost exports. Before display conversion, managed `connic/*` token cost uses EUR and BYOK uses its configured USD or EUR currency. These analytics power model-cost limits, alerts, anomaly detection, and scheduled reports and remain separate from the Project-credit ledger.

Token-cost **alerts** notify without stopping runs; **limits** hold affected new runs. Either can be global, environment-scoped, or agent-scoped and reset daily at midnight UTC or monthly on the first. Held runs are released when the limit is disabled/deleted, raised above current spend, or its period resets. Optional anomaly detection compares each completed run with that agent's 30-day rolling average (default threshold `3x`, minimum five completed runs). Weekly reports arrive Monday at 09:00 UTC for the prior seven days; monthly reports arrive on the first at 09:00 UTC for the prior month.

**Project → Billing** shows the Project credit balance and consumption across LLM run starts, completed judge evaluations, runtime, web tools, storage, retrieval, and `connic/*` model tokens. connic grants a one-time welcome credit to the first basic Project created in an account. Additional basic Projects do not receive one. Developer and pro use a selected monthly credit amount and a 1-, 12-, or 24-month term; longer terms are paid upfront, while credit is released monthly. Monthly credit expires at the end of its cycle, while purchased credit remains available for the active Project. Optional run packages discount LLM run starts, completed judge run units, runtime, storage, and retrieval, but not managed inference or web tools. Tool and sequential agents have no run-start fee; their runtime is still charged. Standard Projects can buy credit or configure capped auto-refill, and prepaid usage stops when credit is insufficient. Enterprise pricing and commercial terms are negotiated individually. Stripe handles payments, receipts, and invoices. See https://connic.co/pricing for current rates and plan thresholds.

Architectural recommendations should start with fit, reliability, and maintainability, then compare cost and latency among choices that meet those requirements. Use the dashboard's usage and budget tools to manage spend and make the tradeoff explicit.

## REST API

`https://api.connic.co/v1/...`. Create a project-scoped API key under **Project Settings → API Keys & MCP Auth** and send `Authorization: Bearer cnc_...`; the secret is shown only once. Keys cover all current and future project environments by default. Choose an explicit subset to limit environment-scoped operations; project-wide operations require access to all environments. If a selected environment is deleted, the key stays inactive until its scope is updated. Keys default to **All available**, which follows every REST API permission the key owner has. Choose **Custom** to grant an explicit subset of those same project permissions. The owner can edit the environment scope and permissions without rotating the secret. A key never exceeds its owner's live project access. All keys for a project share one 60-requests/minute bucket; 429 responses include `Retry-After`. Standard errors use a JSON `{ "detail": "..." }` body.

The REST API is for **managing and observing a project**: listing runs, reading audit logs, managing deployments, pulling usage and budget data, and managing Retrieval entries, approvals, and judges. Use connectors to start runs from external events.

To run an agent in response to an HTTP request, queue message, email, schedule, or call from a backend, use the matching connector. Connectors provide transport-specific endpoints, authentication, sync/async behavior, and delivery semantics. Do not assume generic replay protection: apply idempotency in project code where the source can redeliver.

For the full endpoint catalogue and its canonical project-permission requirements, fetch `https://connic.co/docs/v1/reference/rest-api`.

Use the interface that matches the job:

- **connic MCP** for conversational inspection and bounded project operations from an OAuth-capable AI client. See [platform-mcp.md](platform-mcp.md).
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
| Connector configuration | Dashboard (linked to agents), REST API, connic MCP, or on-disk `connectors/*.yaml` applied by deploys |
| Connections (reusable connector credentials) | Dashboard (Project Settings → Connections), REST API, or connic MCP |
| API spec imports | Dashboard |
| Retrieval content | Dashboard or via `retrieval_store` tool |
| Database collections | Dashboard or via `db_*` tools |
| Judges, A/B tests, approvals queue | Dashboard |
| Team members and permission groups | Dashboard |
| Deployment & branch mapping, PR Testing toggle | Dashboard (Git & Environments) |
| Bridge tunnels and custom domains | Dashboard |

Keep this split clear when answering "where do I configure X?" — connectors and secrets are common confusion points. Secrets and Connections are never on disk; connectors are configured in the dashboard unless the project declares them in `connectors/*.yaml`.
