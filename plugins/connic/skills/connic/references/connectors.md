# Connectors

Connectors define how agents are triggered, what input they receive, and where results go. Most connectors can link to one or more agents; one trigger dispatches its input to every linked agent. Twilio Voice and Telnyx Voice each link to exactly one voice agent. Slack is native. Discord, GitHub, and Notion are not connector types — use a `webhook` connector with your own forwarder, an MCP server, or a custom tool.

Use connectors to run agents from HTTP requests, queue messages, email, schedules, and calls from a backend. They provide provisioned endpoints, transport-specific authentication, sync/async modes, delivery rules, and fan-out. There is no generic inbound deduplication or replay guarantee; design idempotent consumers for transports that can redeliver. The REST API is for project management, not starting event-driven runs.

Connectors are configured per environment in the **Dashboard**, through the REST API or connic MCP, or declared in code as `connectors/*.yaml` and applied by deploys (see [Connectors in code](#connectors-in-code)). They are never configured in agent YAML. For inbound connectors, the incoming event becomes the agent's input. Automatic outbound connectors for Email, Telegram, Slack, Telnyx Messaging, and Twilio Messaging can consume the final agent output; automatic outbound connectors for webhook, Kafka, and SQS publish a full run envelope instead. Agent-tool and middleware outbound connectors use a connector-owned payload schema and do not constrain the final response.

The full list and the modes each one supports:

| Connector | Modes | Use |
| --- | --- | --- |
| `cron` | Inbound | Schedule a recurring run with a fixed prompt. |
| `email` | Inbound (IMAP) / Outbound (SMTP) | Receive emails into the agent; send replies / new mail. |
| `kafka` | Inbound (Consumer) / Outbound (Producer) | Consume from / produce to a Kafka topic. |
| `mcp` | Inbound (Sync / Inbound) | Expose connic agents as MCP tools to external MCP clients. |
| `postgres` | Inbound | LISTEN/NOTIFY-driven trigger. |
| `s3` | Inbound | React to S3 object events (via SNS/EventBridge). |
| `sqs` | Inbound (Consumer) / Outbound (Producer) | Consume from / produce to an SQS queue. |
| `slack` | Inbound (Mentions) / Outbound | Trigger from bot mentions; reply to a thread or post to a channel. |
| `stripe` | Inbound | React to Stripe webhook events. |
| `telnyx_messaging` | Inbound / Outbound | Receive and send SMS/MMS, WhatsApp, and RCS messages. |
| `telnyx` | Sync (incoming voice) | Connect an existing Telnyx phone number to one deployed voice agent. |
| `telegram` | Inbound / Outbound | Telegram bot — receive messages, send replies. |
| `twilio_messaging` | Inbound / Outbound | Receive and send SMS/MMS, WhatsApp, and RCS messages. |
| `twilio` | Sync (incoming voice) | Connect an existing Twilio phone number to one deployed voice agent. |
| `webhook` | Inbound / Outbound / Sync | Generic HTTP. Sync = HTTP request/response; Inbound = fire-and-forget; Outbound = call out to your URL. |
| `websocket` | Sync (real-time chat) | Persistent bidirectional connection; each message is its own run. |

Common dashboard flow: open the agent's detail page → Connector Flow → **Add inbound connector** (inbound and sync types, including voice and WebSocket) or **Add outbound connector** → **Create New Connector** → pick a type → configure → save. Connector names have 1 to 200 characters. A connector's mode can't change while agents are linked ("Unlink the agents before changing the mode."). Kafka, SQS, S3, Postgres, and email connectors reach private endpoints through the **connic bridge** of their Connection: set it on the Connection (dashboard Connection drawer, or the Connection's top-level `bridge` in REST/MCP), never on the connector. An outbound webhook sets `bridge` on the connector. Private MCP servers that an agent consumes use `mcp_servers[].bridge` in agent YAML instead.

At the plan's active connector limit, a new connector is created paused and resuming one returns 403. Telegram, Twilio (voice or messaging), and Telnyx (voice or messaging) connectors change provider settings on create, so at the limit creating one returns 403 instead, and a code deploy that would create or bring back one fails.

File inputs accept up to 50 complete files, with at most 32 MiB combined. Invalid or unavailable optional files are omitted individually. At the first count or size overflow, connic keeps the files already accepted and omits that file and all later files. `file_issues` records omissions; input review can also omit file content that exceeds its budget. Runs can continue with only text and metadata. Under **Linked Agents**, enable **Files required** (link config `files_required: true`) to fail before execution if files are absent or any file is omitted. It's allowed only on types whose events can carry files (`webhook`, `mcp`, `websocket`, `email`, `s3`, `kafka`, `sqs`, `telegram`, `twilio_messaging`, `telnyx_messaging`) and never on outbound connectors; lint, REST, and MCP reject it elsewhere.

## Outbound connector modes

An outbound connector has one mode:

- **Automatic outbound connector** — sends once after a completed run. Choose **All runs**, or choose **Only selected inputs** to send only when `run.connector_id` matches one of the selected inbound or sync connector IDs. All runs also includes manual, cron, and `trigger_agent` runs with no inbound source. Existing links without outbound connector settings remain **Automatic / All runs**.
- **Agent-tool outbound connector** — injects an editable connector tool into an LLM agent (other agent types are rejected). Its default `action_name` is `send_to_<normalized connector name>`. Keep it unique across that agent's custom, predefined, connector, and MCP tools. The model decides whether and when to call it and may call several agent-tool outbound connectors in one run.
- **Middleware outbound connector** — project code calls it with `await send_connector(action_name, payload)`, and the model cannot access it.

Source filters and `StopProcessing(..., publish_outbound=False)` apply only to automatic outbound connectors. The flag does not undo an agent-tool or middleware outbound connector call.

Deployment tests do not deliver outbound messages. Automatic outbound connectors are suppressed, while calls to agent-tool and middleware outbound connectors are recorded as mocked tool calls for assertions.

Each outbound connector owns its payload schema, credentials, destination resolution, wire formatting, retries, and bridge routing. The model and middleware receive no stored secrets or destination values. Agent-tool and middleware outbound connector payloads are:

| Connector | Payload |
| --- | --- |
| HTTP Webhook | `{"payload": { ... }}` |
| Kafka | `{"payload": { ... }, "key": "optional-string"}` |
| SQS | `{"payload": { ... }}` |
| Email | `{"body": "required", "to"?: string \| string[], "subject"?: string, "html_body"?: string, "cc"?: string \| string[], "bcc"?: string \| string[], "reply_to"?: string}` |
| Telegram | `{"text": "required", "chat_id"?: string \| integer}` |
| Slack | `{"text": "required", "channel_id"?: string, "thread_ts"?: string}` |
| Telnyx Messaging | `{"text"?: string, "to"?: string, "media_urls"?: string[], "media_type"?: "image" \| "video" \| "document" \| "audio" \| "sticker", "whatsapp_template"?: object}`; supply text, media, or template content |
| Twilio Messaging | `{"text"?: string, "to"?: string, "media_urls"?: string[], "content_sid"?: string, "content_variables"?: object}`; supply text, media, or template content |

Unknown top-level payload fields are rejected. Routing fields are optional when the connector has a configured default or trusted matching inbound origin. For automatic outbound connectors, the final run output keeps the legacy connector-specific contract documented below. `output_schema` constrains only that final response; it does not combine or replace outbound connector payload schemas.

Custom domains apply to HTTP Webhook, MCP Server, S3, Stripe, and WebSocket connector URLs.

Auth on `webhook`, `websocket`, and `mcp` (server-mode) connectors is governed by a **Require Authentication** toggle (default on). Accepted secret forms differ by transport:

- **Webhook:** `X-Connic-Secret` header (preferred), `Authorization: Bearer`, or `?secret=` query parameter.
- **WebSocket:** either header, `?secret=` / `?X-Connic-Secret=` during the handshake, or `{"secret": "..."}` as the first message.
- **MCP:** `Authorization: Bearer` or `X-Connic-Secret` header; query-string secrets are not accepted.

When off, the connector accepts requests without its shared secret. Verify the caller in `middleware/<agent>.py::before`, for example with a JWT or signed payload (see the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions)).

The same three connectors take per-minute rate limits (`webhook` only in inbound/sync mode), each 0–1,000 with 0 or unset meaning off: total (all senders together), per IP (IPv6 per /64), and per key, where the key is a dot path into the payload (`user_id`, `data.customer_id`) and payloads without the field skip the per-key limit. In `connectors/*.yaml` they go in one block:

```yaml
config:
  rate_limit:      # per minute
    total: 600
    per_ip: 60
    key: user_id
    per_key: 20
```

The REST API and connic MCP take the same `rate_limit` block. The limits count requests for `webhook`, `tools/call` for `mcp` (not `initialize` or `tools/list`; the key path reads the merged `message` + `payload` input), and messages for `websocket` (the key path reads the message `payload`). A rejected request starts no run, never reaches middleware, and uses no Project credit: `webhook` returns 429 with `Retry-After`; `mcp` returns HTTP 429 with a JSON-RPC error carrying `data.retry_after`; `websocket` sends `{"type": "error", "code": "RATE_LIMITED", "retry_after": N}` and keeps the connection open (open connections keep the limits they connected with). The key is unverified sender input, so with **Require Authentication** off a client can rotate or spoof it: use the per-key limit for fairness among trusted senders and the per-IP and total limits against abuse. Provider-pushed and polled connectors have no rate limits.

### Connectors don't have to fire an LLM

Any inbound connector can be linked to a [tool-type agent](agent-yaml.md#tool-agent) instead of an LLM-type agent. The normalized connector payload is passed as one dict to the function's required `payload` parameter, plus `context` when declared — payload keys are never expanded into keyword arguments. The run retains logs, configured retries, and judges without a reasoning step. Use this for deterministic consumers such as Kafka ingestion, S3 transforms, or webhook routing.

## Connectors in code

A project can declare connectors in `connectors/<key>.yaml`. The file name without `.yaml` is the connector's key and identity. Files sit directly in `connectors/` and use `.yaml` (`.yml` and subfolders are rejected). Deploys create, update, and retire these connectors; editing them in the dashboard is disabled.

```yaml
# connectors/order-events.yaml
type: kafka
mode: inbound
name: Order events                # optional, defaults to the file name
connection: orders-kafka-staging  # saved Connection, by name or ID; it holds the Bridge too
config:                           # connector settings, as in the REST API and connic MCP
  topic: ${ORDER_EVENTS_TOPIC}
agents:
  - order-processor
  - name: audit-logger
    files_required: true
environments:                     # omit to deploy to every environment
  Staging: {}
  Production:
    connection: orders-kafka-prod   # another Connection, with the production Bridge
    config:
      group_id: connic-orders-prod
```

Outbound link settings:

```yaml
# connectors/order-results.yaml
type: webhook
mode: outbound
config:
  outbound_url: ${ORDER_RESULTS_URL}
agents:
  - name: order-processor
    outbound: { usage: automatic, source: [order-events] }  # keys of other connector files
  - name: invoice-checker
    outbound: { usage: middleware, action_name: send_invoice_result }
```

File rules:

- **`type`**: any connector type except `slack` (Slack stays dashboard-only). Use `webhook` for HTTP webhooks (`http` also accepted); `twilio` and `telnyx` are the voice connectors. **`mode`** is required when the type has more than one mode; otherwise it defaults to the only one.
- **`name`**: optional display name, 1 to 200 characters; defaults to the file name.
- **`config`**: the connector's settings, the same object the REST API and connic MCP take. `connic lint` and the deploy reject unknown fields, fields that belong to the Connection, fields that don't apply to the mode (`require_auth` or `rate_limit` on an outbound webhook, `group_id` on outbound Kafka, `chat_id` on inbound Telegram, S3 `region` without `include_content: true`, WebSocket `progress_events` with `streaming: false`), missing required fields such as cron `schedule`, and values outside the field's type, options, or range. Lint also checks a cron `schedule` has exactly five fields or is a macro (`@hourly`, `@daily`, `@midnight`, `@weekly`, `@monthly`, `@yearly`, `@annually`; six- and seven-field expressions are rejected), `rate_limit.key` whenever `rate_limit.per_key` is set, numeric Telegram IDs (`allowed_user_ids` entries positive, `chat_id` may be negative), and an outbound webhook `outbound_url` without a `bridge`: `https://`, port 443, no credentials in the URL (whether the host is public is checked when the connector is saved at activation). The file is the connector's full desired state: a key removed from `config` goes back to its default or is cleared, and removing `domain`, or `bridge` from an outbound webhook, detaches the connector.
- **`${VAR}`**: same syntax as `mcp_servers`; resolved once at deploy from the target environment's Variables. Allowed in `config`, `connection`, `bridge`, and `domain`. A missing Variable fails the deploy. The deploy converts each value to its field's type (`true`/`false`, whole numbers) and fails on a value that doesn't fit. Quote phone numbers, sender IDs, and Telegram user IDs in an `allowed_user_ids` list when YAML would read them as numbers (lint: "put it in quotes"); a comma-separated `allowed_user_ids` string also works. Telegram `chat_id` may be an unquoted whole number. Secrets the user supplies (Telegram `bot_token`, Stripe `webhook_secret`, and similar) must be `${VAR}` references; literal values fail the deploy.
- **`connection`**: a saved Connection, by name or ID. Required for `kafka`, `sqs`, `postgres`, `email`, `twilio`, `twilio_messaging`, `telnyx`, and `telnyx_messaging`; `s3` needs one only with `include_content: true`. Their credentials never go in the file. **`bridge`**: Bridge name or ID, for an outbound webhook. Kafka, SQS, S3, Postgres, and email connectors use the Bridge of their Connection; a connector-level `bridge` on them (and S3 `config.endpoint_url`) is deprecated until 2027-04-30, accepted only when it equals the Connection's value (otherwise the deploy fails), and `connic lint` warns about it. Never write it in new files: to use a different Bridge per environment, override `connection` per environment with a Connection that has that Bridge. **`domain`**: hostname or ID of an active custom domain, for types that support custom domains.
- **Generated values** never go in the file, and `connic lint` rejects them: webhook URLs, webhook/WebSocket/MCP secrets, the outbound webhook signing secret, the S3 URL secret, and the Telegram secret token. connic generates them on the first deploy and keeps them; the dashboard shows them as before. An omitted Kafka `group_id` is generated; removing `group_id` from the file keeps the consumer group the connector already uses.
- **Voice** (`twilio`, `telnyx`): set `config.phone_number: "+4930123456"` (E.164, or the number's Twilio SID starting with `PN` or Telnyx ID); the deploy looks an E.164 number up in the Connection. Link exactly one agent with `voice_config`.
- **`agents`**: agent names, or objects with `files_required` and `outbound`. At least one link; every agent must exist in `agents/`. `outbound.usage` is `automatic`, `agent_tool`, or `middleware`; `action_name` is required for `agent_tool` and `middleware`; `source` (automatic only) lists keys of other connector files, each an inbound or sync connector declared in the same environment and linked to the same agent. An agent can use an `action_name` on only one connector per environment, and `agent_tool` needs an `llm` agent; lint or the deploy fails otherwise. `files_required: true` fails lint on outbound connectors and on types whose events carry no files. Without `outbound`, the link is automatic for all runs.
- **`environments`**: keys are environment names, matched exactly. When present, the connector exists only in those environments. Overrides can set `name`, `connection`, `bridge`, `domain`, `agents` (replaces the list), and `config` (merged key by key); `null` removes a `connection`, `bridge`, `domain`, or `config` key set at the top of the file. A key that matches no environment in the project fails the deploy.
- **`apps_only`**: `true` on a sync `webhook` or `websocket` connector (top level only, not per environment) makes it accept only requests connic forwards from the apps that list it in `apps/<key>.yaml`. Its secret and **Require Authentication** no longer apply, and its runs get `context["app"]` and `context["end_user"]`. See [Apps](apps.md#calling-agents-through-connectors).

The REST API (`POST`/`PATCH /v1/projects/{project_id}/connectors`) and connic MCP (`create_connector`, `update_connector`) take the same fields: `type`, `mode`, `name`, `connection`, `bridge`, `domain`, and `config` as an object. Differences: a request names one environment (`environment_id`) instead of `environments`; agents are linked one at a time (`agent_name` on create, the link endpoints, MCP `link_connector`), where the link `config` takes an `agents` entry's `files_required` and `outbound` with `source` as connector IDs; secrets are sent as values (not `${VAR}`) and come back as `********`. Responses return `connection`, `bridge`, and `domain` as IDs; for Kafka, SQS, S3, Postgres, and email connectors, `bridge` is the Connection's Bridge, and sending it back unchanged is fine. An update changes only the fields it sends (a deploy instead makes the connector match its file): a `config` key sent as `null` goes back to its default or is cleared, `domain: null` removes the custom domain, and `bridge: null` removes an outbound webhook's Bridge. In connic MCP, `config` values sent back as `[redacted]` keep their stored values, and errors carry the reason. Changing `mode` returns 400 while agents are linked. Creating a `twilio` or `telnyx` voice connector requires `agent_name` (REST query parameter, MCP argument) naming an agent deployed with `voice_config`. REST can create a `slack` connector once its Slack Connection's setup is ready; MCP and code can't. Older field names (`config.mode`, `config.connection_id`, `config.bridge_id`, `domain_id`, flat `rate_limit_*`, `phone_number_sid`/`phone_number_id`, credentials inline in `config`, link `outbound.version` and `source: {mode, connector_ids}`, MCP `config_json`), a connector-level `bridge`/`config.bridge_id` on Kafka, SQS, S3, Postgres, and email connectors and S3 `config.endpoint_url` (accepted only when they equal the Connection's values, otherwise 400), `config.region` on S3 and SQS Connections (ignored), and creating a connector of a type with several modes without `mode` (falls back to the default mode) still work until 2027-04-30 with `Deprecation`/`Sunset` response headers; never write them in new code.

Lifecycle:

- Every deploy resolves the files for its environment into a snapshot stored with the deployment and logs the planned changes (`create`, `update`, `bring back`, `retire`). Environment names, Connections, Bridges, Variables, phone numbers, `config` values, cron expressions, outbound sources, and action names are checked before the build; errors fail the deploy. Provider setup runs at activation.
- Activating a deployment applies its snapshot: a new deploy, a redeploy of an older commit (Variables resolved fresh), and a rollback to an older deployment (Variables as captured at that deploy). `connic dev` applies the files to its test environment the same way, when the session starts and on every upload that changes them; a change that fails rejects the upload with the reason. Test runs and the deploy gate never change connectors.
- Connectors no longer declared, or every code connector when a deployment has no `connectors/` folder, are **retired**: paused, keeping URL, secrets, links, provider bindings, and run history. Retired connectors don't count against the active connector limit, can't be resumed, and are only deleted by hand. Declaring the same key and type again, or rolling back to a deployment that declared it, brings the connector back with its previous pause state.
- Changing `type` retires the old connector and creates a new one. Renaming the file creates a new connector (new URL and secrets) and retires the old one; change `name:` instead.
- New code connectors start active, including Kafka, SQS, and Postgres. To hold off consuming in an environment, leave it out of `environments:` until ready. Deploys never change the pause state of existing connectors; pause/resume stays in the dashboard. Over the active connector limit, a new connector stays paused, except Telegram, Twilio, and Telnyx connectors: a deploy that would create or bring back one of those at the limit fails.
- Two active connectors (in any environment) can't use the same phone number, messaging sender, or Telegram bot; the deploy fails. A retired connector gives these up when another connector needs them and takes them back if it returns.
- Activation is all or nothing: if a provider call fails, nothing is activated and the previous deployment stays live. **Deploy anyway** on the deployment page redeploys and skips the failing connectors. For a rollback, use **Activate anyway** in the activate dialog (REST: `POST /v1/projects/{project_id}/deployments/{deployment_id}/activate?force=true`). Skipped connectors keep their previous state and are retried on the next activation. The activate dialog lists what a rollback would change.
- A code connector and a dashboard connector can't share a name in one environment; the deploy fails. The dashboard, REST API, and MCP refuse to create or rename a dashboard connector to a name a code connector uses (retired ones included).
- Code connectors show an **In code** badge. Active code connectors cannot be edited, deleted, or linked in the dashboard, and REST/MCP return 409; change the file and deploy instead. Pause/resume stays available.

`connic lint` checks connector files offline: shape, types and modes, `config` fields and literal values, generated fields, literal secrets, `${VAR}` syntax, that linked agents exist, outbound sources and action names, and name clashes between files, and warns about deprecated settings such as a connector-level `bridge` on a Connection type. `connic deploy` runs lint first and uploads `connectors/`.

**Connections** are project-wide credential sets. Manage them under **Project Settings → Connections** (lists the connectors using each Connection per environment), the REST API (`/v1/projects/{project_id}/connections`, permissions `connections.view/create/update/delete`), or connic MCP (`list_connections`, `get_connection`, `create_connection`, `update_connection`, `delete_connection`). A request puts the fields in `config` and the Bridge of a Kafka, Postgres, email, SQS, or S3 Connection in the top-level `bridge` (name or ID; `null` connects directly). Every create and update is checked: missing required fields and unknown fields return 400, and an update is also checked against every connector that uses the Connection (400 naming the connector that couldn't run with the new values). PATCH merges: omitted keys keep their values, `null` removes an optional field (such as `ssl_ca_pem`), and a secret sent as `********` (MCP: `[redacted]`) or empty keeps its stored value. Changing a host, server address, endpoint URL, or the Bridge requires sending the secrets bound to it again (400 otherwise); the dashboard asks for them. Secrets are write-only and masked on read. Slack Connections can only be created in the dashboard. Rotating a Connection updates every connector that uses it without a deploy; deleting one is blocked while a connector uses it.

When a connector is created or updated, from the dashboard or code, connic replaces existing provider-side settings it didn't create and writes the previous values to the connector logs and the project audit log, which keeps them after the connector is deleted (details in each section below). A number, sender, or bot held by another connic connector is still refused.

## cron

Schedule a recurring run.

- **Schedule**: standard five-field cron syntax or a macro such as `@daily` (six- and seven-field expressions with seconds or years are rejected), evaluated in the connector's optional `config.timezone` (an IANA name such as `Europe/Berlin`; UTC by default). Daylight saving time adjusts automatically; `triggered_at` stays in UTC.
- **Prompt**: optional single text prompt (not arbitrary JSON). When configured, it is included in every scheduled payload.
- **Inbound payload shape**: `{"trigger": "cron", "schedule": "<cron expr>", "triggered_at": "<iso>"}`; `prompt` is added only when configured.

## email

IMAP inbound + SMTP outbound. **You bring your own mailbox credentials** — connic does not provision an email address. Inbound and outbound are configured as **separate connectors** (different mode), both linked to the same agent.

connic always verifies the mail server's TLS certificate (no opt-out), including through a Bridge. The certificate must come from a public CA or a private CA in the email Connection's `ssl_ca_pem` (PEM, trusted in addition to public CAs for both IMAP and SMTP), and it must be valid for the IMAP/SMTP host as entered. `ssl_ca_pem` only fixes an unknown CA; for a hostname mismatch, set the host to a name on the certificate (not an IP address). Through a Bridge that name resolves on the customer's network, so internal names work. The error names the failing check.

The email Connection holds IMAP settings, SMTP settings, or both (inbound needs IMAP host, username, and password; outbound needs the SMTP host), the optional `ssl_ca_pem`, and the optional Bridge. SMTP username and password are optional together: leave both empty for a relay without login.

Inbound config: IMAP server / port / username / password on the Connection, mailbox (default `INBOX`), plus optional filters (unread-only, by sender/subject, mark-as-read).

Inbound payload — the keys the agent sees:

```json
{
  "from": "Alice <alice@example.com>",
  "from_address": "alice@example.com",
  "to": "...",
  "subject": "...",
  "date": "...",
  "message_id": "...",
  "body_text": "...",
  "body_html": "...",
  "attachments": [
    {"filename": "...", "content_type": "application/pdf",
     "size_bytes": 12345, "content": "<base64 or text>", "encoding": "base64"}
  ],
  "_email": {
    "connector_id": "uuid-here",
    "mailbox": "INBOX",
    "uid": "12345",
    "timestamp": "2026-07-31T10:30:05.123Z"
  }
}
```

For emails larger than 10 MiB, connic fetches headers, body, and selected attachments separately within bounded limits. Attachment limits apply to decoded content. Supported content includes common images, PDF/text/data formats, and DOCX/XLSX/PPTX; tracking pixels and tiny inline/signature images are filtered out. Omitted attachments have `content: null` and reasons in `file_issues`; by default, the body and metadata still reach the agent even if no attachments are accepted. Field names are `filename`, `content_type`, `content` — not `name`, `mime_type`, `data`.

Automatic outbound connector behavior: configure SMTP server / port / optional username and password on the Connection, From address / From name on the connector, and optionally a Default Recipient. The agent's final *output* is JSON with `to`, `subject`, `body`, and optional `cc`, `bcc`, `html_body`, `reply_to`. A bare string becomes the body. Agent-tool and middleware outbound connectors instead use the Email payload schema above; `body` is required. Recipient precedence is explicit `to`, Default Recipient, then trusted matching inbound email context. Without an explicit subject, replies reuse the inbound subject with `Re:`; other sends use `"Agent Response"`.

## kafka

Inbound (Consumer) and Outbound (Producer) modes.

- **Connection**: a project-wide Kafka Connection holds the bootstrap servers, security protocol, SASL credentials, TLS PEMs, and the optional Bridge. The topic and, for inbound, `group_id` and `auto_offset_reset` go in the connector's `config`. Direct connections require bootstrap and discovered brokers to resolve only to public IP addresses; private brokers require a Bridge on the Connection.
- **Inbound payload**: the parsed message value. Metadata is exposed at `_kafka` inside the payload: `{topic, partition, offset, timestamp, key}` — not in `context`. JSON-object values are dispatched with their top-level fields plus `_kafka`; anything else is wrapped under a `message` key — non-JSON values as `{"message": "<raw text>", "_kafka": ...}`, null values (compaction tombstones) as `{"message": null, "_kafka": ...}`. Tombstones DO trigger runs; use `_kafka.key` to identify the deleted entity.
- **Inbound delivery**: the offset is committed once a queued run is stored for every linked agent, not when the runs finish. If storing fails, the offset stays uncommitted and the same message is redelivered after a short delay, holding back later messages on that connector. Values or keys that are not valid UTF-8 are recorded as failed connector runs and committed. A run that fails later is not redelivered and there is no dead-letter topic; the failed run keeps its input under Runs for re-run. A redelivered message (partial dispatch failure, restart before commit, rebalance) reuses its stored run: each agent gets one run per message and consumer group, also across connectors sharing a Group ID. There is no batch mode.
- **Inbound ordering and parallelism**: a connector reads its partitions one message at a time and stores runs in offset order, but runs execute in parallel up to `max_concurrent_runs`. For per-key ordering, set `concurrency: {key: _kafka.key, on_conflict: queue}` on the agent: same-key runs start one at a time in arrival order, a failed run releases its key, and keyless messages are not serialized. For parallel intake on a high-volume topic, create several inbound connectors with the same Group ID linked to the same agent; Kafka splits the partitions between them.
- **Automatic outbound connector**: publishes a run envelope with `run_id`, `agent_name`, `status`, `output`, `error`, `started_at`, `ended_at`, and `token_usage`. If the run was triggered by inbound Kafka, its key is preserved for partition ordering; otherwise the Kafka key falls back to `run_id`. An agent-tool or middleware outbound connector publishes its `payload` as the message value and may set `key` explicitly.
- **Automatic outbound connector gating** (applies to every outbound connector, not just Kafka): only runs that end `completed` are delivered — failed and cancelled runs are skipped. A run ended via `StopProcessing` counts as completed and is published (the stop message becomes `output`) unless raised with `publish_outbound=False` — see [StopProcessing](tools-and-python.md). Returning `None` or empty output does not suppress the automatic outbound connector.

## mcp (server mode)

Exposes connic agents *as* MCP tools to external clients (Claude Desktop, IDEs, other MCP-aware tools).

- The connector provisions an MCP endpoint URL.
- Each agent linked to the connector becomes one MCP tool. The tool name is lowercased, with spaces and hyphens replaced by underscores; the tool description is `"Invoke the <Agent Name> agent"`. The tool input schema is fixed: `{message: string (required), payload: object (optional)}`. Keys from `payload` are merged into the agent input alongside `message`.
- Attachments use `payload.files` with `name`, `mime_type`, and base64-encoded `data`. Attachment summaries report `accepted`, `omitted`, and `issues` under `structuredContent`. In Sync mode, failed, cancelled, or blocked agent runs return `isError: true`; structured input errors appear under `structuredContent.error`. The complete JSON-RPC body is limited to 16 MiB; larger bodies return HTTP 413 with JSON-RPC error data `code: "request_too_large"`.
- Modes: **Sync** (recommended; returns the agent's result as the MCP tool result, 5-minute timeout) or **Inbound** (returns a run ID immediately; this is connic fire-and-forget behavior, not MCP Tasks).
- Protocols: stateless `2026-07-28` and Streamable HTTP `2025-11-25`, `2025-06-18`, and `2025-03-26`.
- Authentication uses the connector's pre-shared secret in `Authorization: Bearer` or `X-Connic-Secret`; it has no MCP OAuth discovery or interactive authorization flow. Requests with a browser `Origin` header are rejected, so use a native or server-side MCP client.

**Don't confuse this with the `mcp_servers:` block in agent YAML** — that's the opposite direction (connic agent as MCP *client*, calling external MCP tools). See [MCP servers](guardrails-schemas-mcp.md#mcp-servers).

## postgres

**Inbound only**, driven by Postgres `LISTEN/NOTIFY`. The connector subscribes to a channel; each `NOTIFY` becomes one agent run.

- **Config**: the Postgres Connection holds host, port, database, user, password, SSL mode, and the optional Bridge (for private databases). The connector holds the **channel** (the LISTEN channel name) and an optional "Parse JSON Payload" flag.
- **Inbound payload**: includes `_postgres: {channel, pid, timestamp}`. With JSON parsing enabled, an object keeps its top-level fields; a list or scalar is wrapped as `{data: <value>, _postgres: ...}`. With parsing off, invalid JSON, or plain text, it is `{message: <text>, _postgres: ...}`.
- **Payload limit**: Postgres's NOTIFY ceiling is 8000 bytes — design publishers to send a key (e.g. record ID) and have the agent fetch the body via a custom tool.

There are no `postgres.query` / `postgres.fetch_one` tools. To read or write Postgres from inside an agent, write a custom tool with your DB driver of choice (asyncpg, psycopg).

## s3

**Inbound only**, driven by S3 object events. Wire S3 → SNS HTTP subscription → connector URL, **or** S3 → EventBridge → connector URL.

- **Config**: bucket, event mode (**Object Created** by default, or **All Events** to include deletes/restores), optional **prefix/suffix filters**, optional `include_content`, `region` (the bucket's AWS region; required with `include_content: true` and rejected without it in files), and max file size (an integer from 1 to 10 MiB, default 10). Each object event produces one input; content downloads use the configured limit, and omitted content arrives as event metadata with `content.skipped: true` and reasons in `file_issues`.
- **Connection**: needed only with `include_content`. It holds the AWS access key and secret, an optional `endpoint_url` for S3-compatible storage (such as MinIO; scheme, host, and optional port, without credentials, query parameters, or fragments), and an optional Bridge, allowed only together with `endpoint_url`. Direct downloads require public HTTPS; private HTTP(S) endpoints require the Bridge, and a bridged `https://` endpoint needs a certificate from a public CA valid for its hostname (same for a bridged SQS endpoint). Connector-level `config.endpoint_url` is deprecated until 2027-04-30 and accepted only when it equals the Connection's.
- **Inbound payload**: `{bucket, key, size, etag, event_name, event_time, content?, _s3: {event_source, aws_region, request_id, source_ip}}`. When content is included it is `{text, content_type, size_bytes, encoding}`; UTF-8 text uses `encoding: "utf-8"` and binary uses base64.
- **SNS / EventBridge setup**: send to `<connector-url>?secret=<secret-key>`. SNS subscriptions can confirm against this URL; EventBridge can use it as an API Destination.

There are no `s3.get_object` / `s3.put_object` / `s3.list_objects` tools. To upload/list/read from inside an agent, write custom tools using `boto3` / `aioboto3`.

## sqs

Inbound (Consumer) and Outbound (Producer).

- **Connection**: AWS access key ID and secret, an optional `endpoint_url` (an origin for VPC endpoints, ElasticMQ, or LocalStack; without a Bridge it must be public HTTPS), and an optional Bridge. There is no region field: connic reads the region from the endpoint or queue URL host (`us-east-1` when the host names none). Without an endpoint URL, the queue URL must be an AWS SQS queue URL.
- **Inbound config**: queue URL, **visibility timeout** (6–43,200 s, default 300), **max messages** (1–10, default 10), **wait time** (long-polling, 0–20 s, default 20).
- **Inbound payload**: a JSON object keeps its top-level fields; any non-object body is wrapped under `message`. `_sqs: {message_id, receipt_handle, queue_url, approximate_receive_count, sent_timestamp}` is added in both cases.
- **IAM**: inbound consumers need `sqs:ReceiveMessage`, `sqs:DeleteMessage`, and `sqs:ChangeMessageVisibility`; outbound producers need `sqs:SendMessage`.
- **Delivery semantics**: the message is deleted only after **all linked agent runs** succeed; if any fails, it remains for retry. connic tracks dispatched runs and extends message visibility while they are still running. FIFO queues are supported via a Message Group ID.
- **Automatic outbound connector**: sends the same full run envelope as Kafka (`run_id`, `agent_name`, `status`, `output`, `error`, timestamps, and `token_usage`) to the configured queue. An agent-tool or middleware outbound connector sends its nested `payload` as the message body.

## stripe

**Inbound only**. The connector receives Stripe webhook events and verifies the signature.

- **connic side**: configure a connector with a name and a **signing secret** (the `whsec_…` value from Stripe). Without the secret, events are rejected. For a connector defined in code, Edit is disabled: set `webhook_secret: ${VAR}` in its `connectors/*.yaml` file after the first deploy created the URL, and redeploy.
- **Stripe side**: in the Stripe Dashboard, create a webhook pointing at the connector's URL and pick the event types you want. **Event filtering happens in Stripe, not in connic.** The connic connector accepts whatever Stripe sends.
- **Inbound payload**: the parsed Stripe `Event` object.

## telnyx messaging

Use separate inbound and outbound `telnyx_messaging` connectors for SMS/MMS, WhatsApp, or RCS. Both reuse a saved `telnyx` connection containing the account API key and base64-encoded Ed25519 public key. connic can't check on save that the public key belongs to the account (Telnyx has no API for it); copy it from the Telnyx Mission Control Portal (API Keys settings). A wrong key shows up as signature failures in the connector logs.

- **Inbound setup**: choose a channel and sender; the sender picker shows senders the connector can't use as disabled, with the reason. SMS and WhatsApp numbers need a dedicated messaging profile. connic installs the incoming-message webhook on the number's messaging profile or the selected RCS agent, replacing the primary webhook and removing a failover webhook or Telnyx AI assistant; the previous values go to the connector logs and the project audit log and are not restored. An externally hosted WhatsApp number also needs its `whatsapp_profile_id` and manual webhook API version 2 setup in Telnyx.
- **Inbound payload**: includes `text`, `from`, `to`, `channel`, `message_id`, a stable `conversation_id`, and the original Telnyx message payload under `raw`. Downloaded media appears in `files`, with up to 10 attachments and a combined 10 MiB limit. Set `session.key` to `input.conversation_id` when conversations need history; the connector does not enable sessions automatically.
- **Outbound content**: send text, public media URLs, or an approved WhatsApp template. An explicit `to` wins, followed by the configured default recipient and then trusted matching inbound context. A successful connector run records Telnyx's API acceptance and message ID; check Telnyx for final delivery.

Full setup: [Telnyx Messaging documentation](https://connic.co/docs/v1/connectors/telnyx-messaging).

## telnyx voice

Telnyx Voice connects incoming calls on an existing Telnyx number to one deployed agent with `voice_config`. It runs in sync mode and does not place outbound calls. Save the account API key and base64-encoded Ed25519 public key (copy it from the Mission Control Portal; connic can't verify it on save, and signature failures appear in the connector logs), select a voice-capable number (numbers the connector can't use are shown disabled with the reason), and link exactly one deployed voice agent. The agent is required when the connector is created: the dashboard asks for it, REST takes the `agent_name` query parameter, and MCP `create_connector` takes `agent_name`. connic creates a dedicated Call Control application and replaces the number's incoming-call routing, and clears a technical prefix or destination rewrite on the number. The previous values go to the connector logs and the project audit log; removing the connector does not restore them.

Call details: `call_id` is the Telnyx call control ID, `caller_name` comes from `caller_id_name`, and `raw` is the `call.initiated` payload including `custom_headers` and `sip_headers`. Forwarding arrives as a `Diversion` entry in `sip_headers`, so `forwarded_from` is not set.

Optional **Transfer numbers** and **Fallback number** work as for Twilio Voice: `TransferCall` transfers to an allowed number, and only an accepted call the agent can't finish (a failed run, or a media stream that cannot start) goes to the fallback number, or is hung up without one. Calls connic does not answer, such as while the connector is paused, are not forwarded.

Full setup: [Telnyx Voice documentation](https://connic.co/docs/v1/connectors/telnyx-voice).

## telegram

Telegram bot. **Inbound** and **Outbound** are separate connectors (different modes), both backed by the same bot token. An inbound connector replaces a non-connic webhook on the bot and writes the previous URL to the connector logs and the project audit log. `allowed_user_ids` entries must be numeric Telegram user IDs and `chat_id` a numeric chat ID (negative for groups and channels).

- **Inbound**: `message`, `edited_message`, and `callback_query` updates trigger runs.

  Payload shape:

  ```json
  {
    "update_id": 123456789,
    "text": "...",
    "chat_id": 987654321,
    "message": {
      "message_id": 42, "text": "...", "date": 1700000000,
      "chat_id": 987654321, "chat_type": "private",
      "from_id": 987654321, "from_username": "johndoe",
      "from_first_name": "...", "from_last_name": "..."
    },
    "raw": {"update_id": 123456789}
  }
  ```

  Photos, voice messages, audio, videos, video notes, documents, and animations are downloaded when available. File limits apply per message or album; optional omissions still allow an agent run. The largest photo size is used. Each downloaded item appears in top-level `files` as `{name, mime_type, data, size}`, with `data` base64-encoded. Album items are collected until two seconds pass without a new item, then dispatched once per linked agent in message order. The input includes `media_group_id`, `messages`, and `raw_updates`; `text` combines the messages and captions.

  There is no top-level `user_id` — the sender id is `message.from_id`. Optional `Allowed User IDs` allowlist gates which users the bot responds to. Inbound auth is verified by connic via the `X-Telegram-Bot-Api-Secret-Token` header that Telegram sends.

- **Automatic outbound connector**: the final output can be JSON with `text` (also accepts `message` or `body`) and optionally `chat_id`, or a bare string. Agent-tool and middleware outbound connectors use required `text` plus optional `chat_id`. An explicit chat wins, followed by the connector's default Chat ID, then trusted matching inbound Telegram context. Messages use `parse_mode: HTML`.

There is no universal `telegram.send_message` or `telegram.send_photo` predefined tool. An agent-tool outbound connector for Telegram injects only its chosen `action_name`. This outbound connector does not support photos, files, or richer messages; use a custom tool for those Telegram Bot API methods.

## twilio messaging

Use separate inbound and outbound `twilio_messaging` connectors for SMS/MMS, WhatsApp, or RCS. Both reuse saved `twilio` connections with voice: Account SID, region, and regional Auth Token. US1 supports all three channels; IE1 supports SMS text only and excludes +1 senders and recipients; AU1 supports voice only.

- **Config**: required `connection` and `mode`, and in `config` a required `sender` and a `channel` (`sms`, `whatsapp`, or `rcs`; default `sms`). The dashboard's sender picker shows senders the connector can't use as disabled, with the reason. Sender formats are `+E164`, `whatsapp:+E164`, or `rcs:sender-id`; connic adds omitted channel prefixes. Outbound settings optionally include `to`, `messaging_service_sid`, `content_sid`, and `content_variables` (a JSON object).
- **Inbound setup**: connic registers the generated webhook automatically for SMS/MMS numbers and registered WhatsApp or RCS senders, replacing an existing incoming-message webhook or Messaging TwiML App and writing the previous values to the connector logs and the project audit log. It removes the old webhook when the sender changes or the connector is deleted, but only if the webhook still belongs to that connector. The WhatsApp Sandbox requires manual setup: copy the generated URL into **When a Message Comes in**, select HTTP POST, and update or remove it manually later. Messaging setup leaves voice handling unchanged.
- **Inbound payload**: `text`, `from`, `to`, `channel`, `message_sid`, and a stable `conversation_id`. Downloaded media uses standard connic `files` input, limited to 10 MiB total per message.
- **Conversation history**: explicitly configure the agent with `session: {key: input.conversation_id, ttl: 86400}` and redeploy it. The connector does not enable sessions automatically.
- **Outbound content**: automatic delivery accepts a plain string or the structured payload above. Agent-tool and middleware delivery uses the structured payload. Text has a 1,600-character limit; media URLs must be public HTTP/HTTPS URLs without embedded credentials and satisfy the selected channel's media limits.
- **Recipient precedence**: explicit payload `to`, configured default `to`, then the customer from trusted matching inbound context. Reply fallback requires the same saved connection, account, region, channel, and Twilio sender. RCS recipient addresses use `rcs:+E164`, without automatic SMS fallback.
- **WhatsApp templates**: outside the 24-hour customer service window, use an approved `content_sid` (`HX…`) with `content_variables`. Explicit template content cannot be combined with `text` or `media_urls`. A configured Content SID ignores plain generated text; send variables to fill the template.
- **Delivery status**: a completed outbound connector run means Twilio accepted the API request. Check Twilio Messaging Logs for final delivery status.

Full setup: [Twilio Messaging documentation](https://connic.co/docs/v1/connectors/twilio-messaging).

## twilio voice

Twilio Voice is in beta. It connects incoming calls on an existing customer-owned Twilio number to one deployed agent with `voice_config`. It runs in sync mode and does not place outbound calls.

1. Deploy a voice agent using a supported realtime model.
2. In Twilio, open a voice-capable number and confirm its incoming-call region. Remove the number from a SIP trunk if it is on one (Twilio has no API to do this through the number). connic replaces an existing incoming-call webhook or TwiML App and writes the previous values to the connector logs and the project audit log.
3. In connic, create a Twilio connection with the Account SID, region, and that region's Auth Token. Supported regions are US1, IE1, and AU1; US1 is the default. Messaging on the same Connection works in US1 (all channels) and IE1 (SMS text only), not AU1.
4. Add the Twilio Voice connector to the agent, reuse the saved connection, and select the number (numbers the connector can't use are shown disabled with the reason). connic links the agent and installs the number's incoming-call webhook with HTTP POST. The voice agent is required at creation: REST takes the `agent_name` query parameter and MCP `create_connector` takes `agent_name`.

The Auth Token must match the selected region; an API key SID and secret are not accepted in its place. One connection can be reused for other numbers in the same account and region. Changing numbers installs the new webhook and removes connic's webhook from the previous number. Deleting the connector removes the webhook only while it still points to connic.

Call details: `call_id` is the Twilio CallSid; `forwarded_from` comes from `ForwardedFrom`, which Twilio sends only when the caller's carrier passes it; `caller_name` requires caller ID lookup on the number; `raw` holds every parameter of Twilio's incoming-call request (location fields, `StirVerstat`, and so on, where Twilio provides them).

Optional connector settings: **Transfer numbers** (up to 20 E.164 numbers that tools may pass to `TransferCall`) and **Fallback number** (dialed only when an accepted call can't continue, such as a failed run or no free voice capacity; without it the call ends). connic does not change the number's **Primary handler fails** URL. Twilio uses it when connic rejects a call, for example while the connector is paused or the environment has no active deployment; point it at TwiML that dials a staff number to forward those calls. Recorded audio playback is not supported.

Run details show the conversation transcript when transcription is available, tool calls, timings, and errors. Voice configuration and limitations are in [agent-yaml.md](agent-yaml.md#voice-llm-agent).

## slack

Inbound Slack connectors trigger linked agents from signed `app_mention` events. Origin-thread replies require the inbound and outbound connectors to use the same verified Slack Connection.

Configure Slack in the Dashboard: create a Slack Connection, open the generated app manifest, save the Signing Secret, verify the Events request URL, install the app, and save its `xoxb-` Bot User OAuth Token. Then create the inbound or outbound connector and invite the bot to every channel where it should receive mentions or post messages. connic MCP can read Slack connectors but cannot create them; the REST API can once the Slack Connection's setup is ready. Slack connectors can't be declared in code.

One Slack Connection can have one inbound connector across all environments. Link additional agents to that connector; outbound connectors can reuse the Connection. To receive mentions in another environment, create a second Slack app (a new Slack Connection). While a Slack Connection isn't ready (after its signing secret was replaced and the Events URL isn't verified again, or after the app was uninstalled), automatic outbound deliveries are recorded as failed connector runs with `slack_setup_not_ready`.

- **Inbound payload**: includes `text`, `user_id`, `team_id`, `channel_id`, `thread_ts`, and `event_id`.
- **Automatic outbound connector**: a plain final response is sent as text; JSON may use `text`, `message`, or `body` plus optional `channel_id` and `thread_ts`.
- **Agent-tool / middleware outbound connector**: `text` is required; `channel_id` and `thread_ts` are optional. An explicit route wins, then trusted origin context from an inbound connector using the same Slack Connection, then the configured default channel.
- Connection credentials, origin routing, rate-limit handling, and retries stay connector-owned.

## webhook

The most flexible HTTP connector. Three independent modes — pick the one that matches your traffic pattern.

| Mode | What it does |
| --- | --- |
| **Sync (Request-Response)** | Caller `POST`s; connector blocks until the agent finishes and returns the result. 5-minute hard timeout. |
| **Inbound (Fire & Forget)** | Caller `POST`s; connector returns immediately with `{status, dispatched_to, run_ids[]}`. |
| **Outbound** | Automatic outbound connectors POST the full completed-run envelope. Agent-tool and middleware outbound connectors POST their nested `payload`. Both use the configured URL and signature headers. |

Inbound and Sync also accept `GET` (query params become the payload, with the authentication `secret` stripped), `application/x-www-form-urlencoded`, and `multipart/form-data` (uploaded files are passed to the model; whether it can read a file type depends on the model, see [Models](platform.md#models)). The request body has room for 32 MiB of files with their encoding, plus metadata; larger bodies return HTTP 413 with `code: "request_too_large"`.

For multipart, `context["payload"]` is normalised to the same shape `trigger_agent` uses for [passing files](predefined-tools.md#passing-files-to-the-triggered-agent):

```python
{
  "customer_id": "cus_123",   # multipart text fields sit at the top level
  "tag": ["a", "b"],          # repeated keys are grouped into a list
  "message": "...",
  "files":   [{"name": "invoice.pdf", "field_name": "file",
               "mime_type": "application/pdf",
               "data": "<base64>", "size": 12345}]
}
```

Form fields and any `file_issues` remain available when no files are accepted. Multipart file parts use a fixed allowlist (images, PDF, text/CSV/JSON/XML, Office, ODF, EPUB). JSON requests can supply `files` with `name`, `mime_type`, and base64-encoded `data`. Both follow the file limits and **Files required** setting above.

The LLM-facing `content` is reconstructed automatically: each `files[*]` entry becomes a binary part, and the leading text part is the payload with only `files` removed. A `{message, files}` payload renders as the plain `message` string; any richer shape is JSON-serialised.

**Authentication** (Inbound / Sync) is controlled by a **Require Authentication** toggle on the connector. When on (the default), callers must present the connector secret as `X-Connic-Secret: <secret>`, `Authorization: Bearer <secret>`, or `?secret=<secret>`; unauthenticated requests are rejected. When off, verify the caller in `middleware/<agent>.py::before`, for example with a JWT or signed payload (see the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions)).

**Verifying the sender.** Webhook runs carry the HTTP request in `context["request"]`: `method`, `headers` (lowercase names), `query`, `received_at` (when connic received the request, UTC ISO 8601), and the raw body as `body_base64`. connic removes its own secret (`X-Connic-Secret`, a matching `Authorization`, the `secret` query parameter) and headers its network adds. The body is omitted for multipart requests and above 1 MiB (`body_omitted: true`). Run details show the body and the `Authorization`, `Proxy-Authorization`, `Cookie`, and `X-API-Key` headers as `[REDACTED]`. Runs started another way (reruns, tests, `trigger_agent`) have no `request`. The check runs when the run starts: a failing request still creates a run that middleware stops. Because a run can start well after its request arrived, check token expiry (`exp`/`nbf`) against `received_at` with a few minutes of leeway, not the current time.

HMAC signature, such as GitHub's `X-Hub-Signature-256`:

```python
import base64
import hashlib
import hmac
import os

from connic import StopProcessing


async def before(content, context):
    request = context.get("request")
    if request is None:  # Not started by a webhook request, such as a rerun
        return content
    body = base64.b64decode(request.get("body_base64") or "")
    digest = hmac.new(os.environ["GITHUB_WEBHOOK_SECRET"].encode(), body, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(f"sha256={digest}", request["headers"].get("x-hub-signature-256", "")):
        raise StopProcessing("Invalid signature", publish_outbound=False)
    return content
```

Signed JWT, such as the Bot Framework token Microsoft Teams sends (`PyJWT[crypto]` in `requirements.txt`; Teams can't send a secret header, so append `?secret=` to the messaging endpoint URL):

```python
import asyncio
import os
from datetime import datetime

import jwt
from connic import StopProcessing

keys = jwt.PyJWKClient("https://login.botframework.com/v1/.well-known/keys")
LEEWAY = 300  # Seconds of clock skew


async def before(content, context):
    request = context.get("request")
    if request is None:
        return content
    token = request["headers"].get("authorization", "").removeprefix("Bearer ")
    try:
        key = await asyncio.to_thread(keys.get_signing_key_from_jwt, token)
        claims = jwt.decode(
            token, key.key, algorithms=["RS256"], audience=os.environ["MICROSOFT_APP_ID"],
            issuer="https://api.botframework.com", leeway=LEEWAY,
            options={"require": ["exp"], "verify_exp": False, "verify_nbf": False},
        )
    except jwt.PyJWTError:
        raise StopProcessing("Invalid token", publish_outbound=False)
    received_at = datetime.fromisoformat(request["received_at"]).timestamp()
    if not claims.get("nbf", 0) - LEEWAY <= received_at <= claims["exp"] + LEEWAY:
        raise StopProcessing("Invalid token", publish_outbound=False)
    if claims.get("serviceUrl") != context["payload"].get("serviceUrl"):
        raise StopProcessing("Invalid token", publish_outbound=False)
    return content
```

Sync example (auth enabled):

```bash
curl -X POST <webhook-url> \
  -H "Content-Type: application/json" \
  -H "X-Connic-Secret: <secret>" \
  -d '{"question": "What is the capital of France?"}'
```

Sync example (auth disabled, JWT verified in middleware):

```bash
curl -X POST <webhook-url> \
  -H "Content-Type: application/json" \
  -d '{"auth_token": "<jwt>", "question": "What is the capital of France?"}'
```

Sync response:

```json
{
  "status": "ok",
  "result": {
    "run_id": "...",
    "agent_name": "intake-classifier",
    "status": "completed",
    "output": "Paris",
    "error": null
  }
}
```

The URL itself is provisioned per connector — copy it from the connector's detail drawer in the dashboard. Don't hard-code an assumed URL format.

For sync, the agent's response (string or — if `output_schema` is set — structured object) is what populates `result.output`. For inbound, the response is the dispatch confirmation; the run's output is visible in the dashboard. An automatic outbound connector sends the same run envelope as Kafka and SQS (`run_id`, `agent_name`, `status`, `output`, `error`, timestamps, and `token_usage`); agent-tool and middleware outbound connectors send their nested `payload`. Verify the request's hex HMAC-SHA256 over `timestamp + "." + raw_body` using the connector signing secret and a constant-time comparison; reject timestamps outside a five-minute window.

## websocket

A single "Sync (Real-time Chat)" mode. The connector hosts a WS endpoint; each connection carries messages, and every message starts its own agent run.

- **Auth**: governed by the same **Require Authentication** toggle as the webhook connector (default on). When on, send `{"secret": "<connector secret>"}` as the first message after connecting, or pass `X-Connic-Secret` as a query param / header during the handshake. When off, the WS endpoint is open and authentication is your responsibility: typically a per-user JWT in every message payload, verified in `middleware/<agent>.py::before`. A token sent only in the first message is not available to later runs. Turn it off when each connection already carries a stronger per-user credential than a shared secret would provide.
- **Message protocol**: client sends the canonical `{type: "message", id?, payload: {message, context}}` envelope; shorthand `{"message": "..."}` and `{"content": "..."}` forms are also accepted. Server replies `ack` → `stream_start` → `stream_chunk` (multiple) → `stream_end` (with `full_response`, `token_usage`) when streaming is on; or a single `response` message when streaming is off. Agents with output guardrails still use the streaming event contract, but send one `stream_chunk` after the run completes so guardrails can inspect the full response before any text is released.
- **Progress events** (opt-in **Progress Events** setting, streaming only): `thinking` when a model call starts, `tool_call` / `tool_result` with the raw `tool` name (the target tool for `use_tool`; never arguments or results), and `approval_pending` / `approval_resolved` (`decision`: `approved`, `rejected`, `timeout`) around an approval pause. Each carries the message `id` and `agent`. They also arrive for agents with output guardrails. The client chooses the labels.
- **Files / multimodal**: the payload may carry a top-level `files` array in the same shape as the [webhook multipart normalization](#webhook) (`{name, mime_type, data: "<base64>", size}`); each accepted entry becomes a binary part of the LLM-facing `content`. There is no fixed MIME allowlist, but the file limits and **Files required** setting above apply.
- **Ordering**: a connection handles one message at a time, in arrival order. A message sent while a response is running waits until all linked agents have responded; there is no busy rejection. A response can take as long as the run timeout; approval waits don't count, so an approval-gated tool holds the turn until decided. Keep `approval.timeout` short on chat agents. Runs from different connections are not serialized; see the session concurrency note in [agent-yaml.md](agent-yaml.md#sessions).
- **Config**: streaming toggle, progress events toggle (default off; applies only with streaming on), session timeout (idle seconds before the connection closes, 60–86,400, default 3,600), max messages per connection (1–10,000, default 100), rate limits (total, per IP, per key; see above). Values outside these ranges are rejected, not clamped.
- **`connector_run_id`** is returned on connect and identifies the connector run that records the connection.
- **Conversation history** comes only from the agent's `session:` config, not from the connection. Set `session.key: input.conversation_id` (or a `context.*` key set in middleware) and send that field in every message; without a key, all conversations share one session. History outlives the connection: reconnecting with the same ID continues it.

## Pausing a connector

Pause a connector from its drawer in the Connector Flow (**Pause** / **Resume**) or with connic MCP `update_connector` and `enabled: false`. A paused connector starts no new runs; runs already queued or running are not affected. What happens to events during the pause depends on the type:

- **Kafka, SQS**: connic stops reading. Messages stay in the topic or queue and are processed after resume. Kafka continues from the consumer group's last committed offset (offsets are committed after the run is created), so messages must still be within topic retention.
- **Postgres**: connic stops listening. Notifications sent during the pause are lost.
- **Webhooks and messaging** (such as `webhook`, `stripe`, `slack`, `telegram`, `twilio_messaging`): requests are acknowledged without starting runs. The events are not stored or replayed.
- **WebSocket**: new connections are refused; an open connection gets a `CONNECTOR_INACTIVE` error on its next message and closes.
- **Twilio Voice, Telnyx Voice**: incoming calls are rejected.
- **Cron**: no runs start; schedule times missed during the pause are not made up.

New inbound Kafka, SQS, and Postgres connectors created in the dashboard start paused; resume them once linked. Connectors declared in `connectors/*.yaml` start active. To stop intake during an incident or maintenance window without losing events, pause a Kafka or SQS connector.

## Linking a connector to multiple agents

Connectors that support multiple agents trigger every linked agent in parallel for each event. This supports fan-out, such as one webhook starting both an `intake` agent and an `audit` agent. Each agent runs independently with its own input, logs, and result. Twilio Voice and Telnyx Voice are single-agent connectors and do not support fan-out.

## Where the payload ends up

When a connector fires, the inbound event reaches `middleware/<agent>.py::before(content, context)` in two forms:

- **`content`** — the LLM-facing message: a dict with `role: "user"` and a list of `parts` (text and/or binary attachments). This is what the agent reasons over. Mutate it to attach documents, prepend context, redact PII.
- **`context["payload"]`** — the connector's normalized payload before user middleware transforms it: the JSON body of a webhook, GET query params, normalized multipart fields/files, parsed Kafka message plus metadata, email fields, etc. Read it for auth tokens, identity claims, routing metadata, and anything else you want middleware to see but the LLM should not.

Per-connector payload shapes are documented in the sections above. Normalize them in middleware when tools need one stable schema. See [the authentication walkthrough](tools-and-python.md#end-user-authentication-and-per-run-permissions) for using credentials from `context["payload"]`.
