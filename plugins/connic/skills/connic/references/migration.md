# Migrating an existing app to connic

Use this when moving an existing codebase to connic: a prototype, a set of scripts, a web app that calls a model, or an agent framework project. The result should be a connic project that uses connic's own triggers, state, orchestration, safety, and operations. Re-hosting the old app behind a custom tool is not a migration.

## Runtime facts that decide most choices

- **Files written at runtime are temporary.** They can disappear between runs and are reset on every deploy. Use disk only for scratch files inside one run. A SQLite file, JSON file, pickle, or local vector index that the app writes to does not survive on connic.
- **Module-level variables are not storage.** They can cache clients (an HTTP or JWKS client) but never hold records, counters, or conversation history.
- **Custom code is Python.** Tools, middleware, hooks, and guardrails are Python modules. Port JavaScript or TypeScript agent logic to Python instead of keeping a Node service running for tools to call.
- **connic runs the agent loop.** A hand-written loop that calls the model, parses tool calls, and dispatches functions is dropped, not ported.
- **Each environment is separate.** Database, Retrieval, sessions, Variables, and connectors differ per environment.
- **Schedules run in UTC.** Convert local cron times.

## What moves and what stays outside

connic hosts the agent side: model calls, tools, triggers, schedules, and the state agents read and write. Two things stay where they are and call connic through connectors:

- **A frontend** (web pages, mobile app) calls a `webhook` (Sync) or `websocket` connector. A browser must never hold the connector secret: turn **Require Authentication** off and verify an end-user token in `middleware/<agent>.py::before` ([pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions)), or call through the app's own backend.
- **Systems of record** that other software also uses (the company database, CRM, internal APIs) stay and are reached from custom tools.

Everything the app owns moves.

### Keep-or-move test for every data store

Ask: **does anything besides this app read or write this store?**

- **No: it is app data, so move it.** Records go to the connic database through wrapped `db_*` tools. Documents the app searches by meaning go to Retrieval. Conversation history goes to [`session:`](agent-yaml.md#sessions). Load existing data once (see [Moving existing data](#moving-existing-data)).
- **Yes: it is a system of record, so keep it.** Reach it from a custom tool with its driver in `requirements.txt`. If the app reacted to changes in it, use the `postgres` connector (LISTEN/NOTIFY) or a queue connector. Use a [Bridge](platform.md#bridge) only when the store runs on always-on private infrastructure such as a VPC or an on-prem network.

These are never valid outcomes:

- A bridge to `localhost` or a developer machine. Production would depend on that machine staying on.
- A file-based store (SQLite, JSON, pickle, local vector index) kept inside the project.
- The old app running next to connic so tools can call it.
- Keeping something because moving it is more work.

Database queries address one collection per call. Store rows the app reads together as one document instead of joining tables. If the app depends on joins or transactions across tables, say so in the plan and let the user decide between reshaping the data and a managed external database. A local file is still not an option.

## Workflow

1. **Inventory, read-only.** Walk the whole source tree and list every entry point, model call, prompt, function the model calls, data store, schedule, background job, external integration, secret, and auth check. The [pattern map](#pattern-map) lists what to search for.
2. **Map every item.** Give each one an outcome: **native** (which connic feature), **external** (which system, and why it passes the keep-or-move test), or **dropped** (replaced by connic, such as the agent loop or the HTTP server). Include the model choice: an exact `connic/*` ID from the [connic model catalog](https://connic.co/docs/v1/build/connic-models), or BYOK to keep the current provider model unchanged. Don't choose silently.
3. **Show the plan before writing files.** Present the mapping, the kept-external items with reasons, and what the user must do (Variables, Connections, Bridges, data import). Wait for confirmation.
4. **Build in a new directory.** Leave the old app untouched for comparison. Scaffold with `connic init`, then write agents, tools, middleware, schemas, guardrails, tests, and `connectors/*.yaml` ([connectors in code](connectors.md#connectors-in-code); Slack stays in the dashboard). Migrated agents follow the same [best practices](../SKILL.md#best-practices-apply-these-by-default) as new ones.
5. **Validate** with `connic lint`. Ask before `connic dev`, `connic deploy`, or a data import; each acts on the user's connic account.
6. **Move data** into each environment.
7. **Write `MIGRATION_REPORT.md`.**

For a LangChain or Google ADK project, run [`connic migrate`](cli-and-dev.md#connic-migrate) first. Then apply this workflow to everything its report lists as manual, and extend its `MIGRATION_REPORT.md` instead of replacing it.

## Pattern map

The left column is what to look for in the source; the right column is what replaces it.

### Entry points

| In the source | On connic |
| --- | --- |
| HTTP route that calls a model (FastAPI, Flask, Express, Next.js API route) | `webhook` connector: Sync (5-minute limit) or Inbound (fire-and-forget) |
| Streaming chat endpoint (SSE, WebSocket server) | `websocket` connector with streaming |
| Telegram, Slack, or email handlers (IMAP polling, SMTP replies) | `telegram`, `slack`, or `email` connector |
| SMS or WhatsApp webhook handler | `twilio_messaging` or `telnyx_messaging` connector |
| Phone-call handling (media streams, speech-to-text and text-to-speech glue) | Voice agent (`voice_config`) with a `twilio` or `telnyx` connector |
| Stripe webhook handler | `stripe` connector |
| Kafka or SQS consumer script, function triggered by S3 events | `kafka`, `sqs`, or `s3` connector; link a [`tool` agent](agent-yaml.md#tool-agent) when no model is needed |
| Discord bot, GitHub webhook | `webhook` connector with a forwarder, an MCP server, or a custom tool (no native connector) |
| The app exposes its agent as an MCP server | `mcp` connector |
| Auth check in the route (API key, JWT, session) | Connector secret for service callers; end-user tokens verified in `before` middleware, with conditional tools gating permissions |

### Schedules and background work

| In the source | On connic |
| --- | --- |
| `schedule`, APScheduler, `node-cron`, crontab, scheduled CI jobs, `while True` loops with `sleep` | `cron` connector (UTC) |
| Delayed job, `setTimeout`, Celery `countdown` or `eta`, "remind in 3 days" | `trigger_agent_at` (up to 30 days; beyond that, a cron connector with self-rescheduling) |
| Internal job queue (Celery, RQ, BullMQ) passing work to the next step | `trigger_agent` with `wait_for_response=False` |
| Queue shared with other systems | `kafka` or `sqs` connector, inbound and outbound |
| Per-user locks, one job per customer at a time, debouncing | `concurrency.key` with `on_conflict: queue` or `coalesce` |
| Polling a table for new rows | If the store moved: call `trigger_agent` from the tool that writes the row. If it stayed in Postgres: `postgres` connector |

### Model logic

| In the source | On connic |
| --- | --- |
| Chat-completion call with a system prompt | LLM agent with `model` and `system_prompt` |
| Prompt assembled from request data with f-strings or templates | `{var}` interpolation from `context`, set in `before` middleware |
| Tool or function JSON schemas plus a dispatch loop | `tools/*.py` functions with type hints and docstrings; the loop is dropped |
| Fixed chain of model calls | `sequential` agent, with deterministic steps as `tool` agents |
| Router that classifies, then calls a different prompt | Orchestrator agent that calls specialist agents with `trigger_agent` |
| `json.loads` on the response plus retry on parse errors | `output_schema` with `temperature: 0` |
| Retry and backoff wrappers, switching models on failure | `retry_options`, `fallback_model` |
| Trimming or summarizing history to fit the context window | `context_compression` |
| Text extraction from PDFs before the model call | Pass the file; models with the Files capability read PDFs natively ([platform.md](platform.md#models)) |
| A large set of rarely used tools | `discoverable_tools` |

### State and data

| In the source | On connic |
| --- | --- |
| `sqlite3`, an ORM on a local file, `lowdb`, JSON or pickle files, module-level dicts holding records | Database collections through wrapped `db_*` tools; limit each agent's collections with the `database:` block |
| Chat history in a list, table, or Redis | `session:` with a `key` such as `input.conversation_id` |
| Vector store the app owns (Chroma, FAISS, LanceDB, its own pgvector table) plus chunking and embedding code | Retrieval namespaces behind a wrapped `retrieval_query`; the chunking and embedding code is dropped |
| Content pulled from Notion, Confluence, Zendesk, SharePoint, OneDrive, or a website | Retrieval sources with scheduled sync |
| A database other systems use | Stays; custom tool, plus a bridge if it is private |

### Integrations

| In the source | On connic |
| --- | --- |
| Sending email | `email` outbound connector |
| Posting to Slack, Telegram, SMS, or WhatsApp | Outbound connector (automatic, agent-tool, or middleware) |
| POSTing results to another service | `webhook` outbound connector, or a Kafka or SQS outbound connector |
| Web search API client | `web_search` |
| Scraping (requests with BeautifulSoup, Playwright, Selenium) | `web_read_page`, `web_browser_*` |
| Public REST API with an OpenAPI spec | API spec tools |
| Third-party tools available as an MCP server | `mcp_servers:`; per-user tokens through OAuth apps |
| Private internal API | Custom tool through a bridge |

### Safety, quality, and operations

| In the source | On connic |
| --- | --- |
| Moderation calls, keyword or regex filters, prompt-leak checks | Guardrails: `prompt_injection`, `pii`, `moderation`, `regex`, `topic_restriction`, `system_prompt_leakage` |
| "Ask a human if the amount is over X", admin review queues | `approval.tools` with `param.*` conditions |
| Asking the user for a code or confirmation text mid-run | `approval.inputs` |
| Logging, tracing, and cost tracking code or services | Built-in run traces, logs, and dashboards; set `context["customer_id"]` in middleware for cost per customer |
| Eval scripts, golden datasets | `tests/*.yaml` as the deploy gate, plus judges |
| Randomly assigned prompt variants | A/B test variants (`<agent>-test-<variant>.yaml`) |

### Hosting and configuration

| In the source | On connic |
| --- | --- |
| `.env`, `os.environ`, `process.env` | Variables per environment, secrets marked Sensitive; connector credentials in Connections |
| Model provider API key | Not needed for `connic/*`; BYOK credentials in **Project Settings** |
| Dockerfile, Procfile, server entry point (uvicorn, gunicorn), health checks, platform deploy files | Dropped; deploy through Git or `connic deploy` |
| Staging and production config switches | Environments |
| `requirements.txt`, `package.json` | `requirements.txt` with only what tools import, never `connic-composer-sdk` |

## Moving existing data

- **Records.** Export each app-owned table or file as a JSON array of objects or as JSONL, one file per collection, already shaped as documents. connic assigns its own `_id`, `_created_at`, and `_updated_at`, so keep old IDs and references between rows in a regular field such as `legacy_id`. Each collection is imported once, all or nothing, into a new or empty collection. An import into a collection that already has documents fails with 409, so an import cannot be split into chunks; to redo one, delete the collection and import again.
- **Import path.** With connic MCP connected, `import_database_documents` takes up to 512 KiB of documents per collection. Larger files, up to 25 MB, go through **Import** on the dashboard's Database page or a multipart `file` upload to `POST /v1/projects/{project_id}/database/{environment_id}/collections/{name}/import`. All three paths share a limit of 50 import requests per environment in any rolling 24 hours, failed ones included; beyond it the response is 429 with `Retry-After` in seconds. Check every export file is a valid JSON array or JSONL before importing rather than retrying failed imports.
- **Documents.** Upload the original files to Retrieval, or connect a Retrieval source. Don't carry over stored vectors.
- **Environments.** Import into one environment, then use the Database and Retrieval copy drawers to copy into others that need the same data. Staging usually gets test data rather than production records.

## `MIGRATION_REPORT.md`

Write it at the root of the new project:

- **Mapping:** each source component, the connic feature that replaces it, and the files involved.
- **Kept external:** each system, why it stays, and how agents reach it.
- **Dropped:** what connic replaces, such as the agent loop, HTTP server, scheduler, and vector-store code.
- **User actions:** Variables per environment, Connections and bridges to create, Slack setup in the dashboard, data imports per environment, and callers to point at connector URLs.
- **Behavior changes:** a different model, UTC schedules, new payload shapes, anything else that differs from the old app.

## Example

A FastAPI app: `POST /chat` calls a model and keeps history in SQLite, with a `tickets` table in the same file. APScheduler sends a nightly digest to a Slack webhook. Product docs sit in a local Chroma index, and `.env` holds the model key and the Slack URL.

| Source | connic |
| --- | --- |
| `POST /chat` | `connectors/chat.yaml` (webhook, Sync) linked to `agents/assistant.yaml` |
| `messages` table | `session` with `key: input.conversation_id` on `assistant` |
| `tickets` table | Database collection `tickets`, `tools/tickets.py` with `find_open_tickets` and `create_ticket`, imported once per environment |
| APScheduler nightly job | `connectors/nightly-digest.yaml` (cron, UTC) linked to `agents/digest.yaml` |
| Slack webhook post | Slack outbound connector on `digest`, created in the dashboard |
| Chroma index and chunking code | Retrieval namespace `docs`, `tools/docs.py::search_docs` wrapping `retrieval_query`, source files uploaded again |
| Model key in `.env` | A `connic/*` model, or BYOK in **Project Settings** |
| FastAPI, uvicorn, APScheduler, chromadb, the SQLite file, Dockerfile | Dropped |
| Web frontend | Stays; calls the `chat` connector with an end-user token verified in middleware |
