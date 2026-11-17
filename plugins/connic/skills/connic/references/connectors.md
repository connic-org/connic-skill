# Connectors

Connectors define how agents are triggered, what input they receive, and where results go. Most connectors can link to one or more agents; one trigger dispatches its input to every linked agent. Twilio Voice, Telnyx Voice, and SIP Voice each link to exactly one voice agent. Slack, Microsoft Teams, GitHub, GitLab, Jira, Linear, Notion, Pipedrive, Shopify, and WhatsApp are native. Discord is not a connector type — use a `webhook` connector with your own forwarder, an MCP server, or a custom tool.

Use connectors to run agents from HTTP requests, queue messages, email, schedules, and calls from a backend. They provide provisioned endpoints, transport-specific authentication, sync/async modes, delivery rules, and fan-out. There is no generic inbound deduplication or replay guarantee; design idempotent consumers for transports that can redeliver. The REST API is for project management, not starting event-driven runs.

Connectors are configured per environment in the **Dashboard**, through the REST API or connic MCP, or declared in code as `connectors/*.yaml` and applied by deploys (see [Connectors in code](#connectors-in-code)). They are never configured in agent YAML. For inbound connectors, the incoming event becomes the agent's input. Automatic outbound connectors for Email, Outlook, Gmail, Google Sheets, Telegram, Slack, Microsoft Teams, GitHub, GitLab, Jira, Linear, Notion, Shopify, Telnyx Messaging, Twilio Messaging, and WhatsApp can consume the final agent output; automatic outbound connectors for webhook, Kafka, SQS, RabbitMQ, MQTT, Azure Service Bus, and Google Pub/Sub publish a full run envelope instead. Agent-tool and middleware outbound connectors use a connector-owned payload schema and do not constrain the final response.

The full list and the modes each one supports:

| Connector | Modes | Use |
| --- | --- | --- |
| `azure_service_bus` | Inbound (Receiver) / Outbound (Sender) | Receive from an Azure Service Bus queue or topic subscription / send to a queue or topic. |
| `cron` | Inbound | Schedule a recurring run with a fixed prompt. |
| `email` | Inbound (IMAP) / Outbound (SMTP) | Receive emails into the agent; send replies / new mail. |
| `github` | Inbound (Repository events) / Outbound | Trigger from issue and pull request events; comment on the issue or pull request or open an issue. |
| `gitlab` | Inbound (Project events) / Outbound (Replies) | Trigger from issue, merge request, and comment events in a GitLab project (gitlab.com or self-managed); comment on the issue or merge request that started the run. |
| `gmail` | Inbound / Outbound | Receive email from a Google Workspace mailbox; reply in the thread or send new mail through the customer's service account. |
| `google_pubsub` | Inbound (Push subscription) / Outbound (Publish) | Trigger from the messages a Google Cloud Pub/Sub push subscription delivers, authenticated by Google's OIDC token; publish results to a topic. |
| `google_sheets` | Inbound / Outbound | Trigger from new rows in a spreadsheet; append results as rows through the customer's service account. |
| `jira` | Inbound (Issue events) / Outbound | Trigger from Jira Cloud issue and comment events; comment on the issue or create an issue. |
| `kafka` | Inbound (Consumer) / Outbound (Producer) | Consume from / produce to a Kafka topic. |
| `linear` | Inbound (Team events) / Outbound | Trigger from issue and comment events in a Linear team; comment on the issue (in the comment's thread) or create an issue. |
| `mcp` | Inbound (Sync / Inbound) | Expose connic agents as MCP tools to external MCP clients. |
| `mqtt` | Inbound (Subscriber) / Outbound (Publisher) | Subscribe to an MQTT 5 topic filter / publish to a topic. |
| `notion` | Inbound (Page and comment events) / Outbound | Trigger from new and changed pages and new comments in Notion; comment on the page (in the comment's discussion) or create a page in a data source. |
| `outlook` | Inbound / Outbound | Receive email from a Microsoft 365 mailbox; reply in the thread or send new mail through the customer's Entra ID app. |
| `pipedrive` | Inbound (Record events) / Outbound (Notes) | Trigger when deals, leads, people, organizations, activities, or notes are created, updated, or deleted; add a note to the record. |
| `postgres` | Inbound | LISTEN/NOTIFY-driven trigger. |
| `rabbitmq` | Inbound (Consumer) / Outbound (Publisher) | Consume a RabbitMQ queue / publish to an exchange. |
| `s3` | Inbound | React to S3 object events (via SNS/EventBridge). |
| `shopify` | Inbound (Order events) / Outbound (Tags and notes) | Trigger from order events in a Shopify store through the customer's own Dev Dashboard app; add or remove tags on the order or add to its note. |
| `sip` | Sync / Outbound (voice) | Answer and place calls through a SIP phone system or provider. |
| `sqs` | Inbound (Consumer) / Outbound (Producer) | Consume from / produce to an SQS queue. |
| `slack` | Inbound (Mentions) / Outbound | Trigger from bot mentions; reply to a thread or post to a channel. |
| `stripe` | Inbound | React to Stripe webhook events. |
| `teams` | Inbound (Messages) / Outbound | Microsoft Teams through the customer's single-tenant Azure Bot: trigger from chats with the bot and mentions in channels and group chats; reply in the conversation or post to a channel. |
| `telnyx_messaging` | Inbound / Outbound | Receive and send SMS/MMS, WhatsApp, and RCS messages. |
| `telnyx` | Sync / Outbound (voice) | Answer and place calls from an existing Telnyx phone number. |
| `telegram` | Inbound / Outbound | Telegram bot — receive messages, send replies. |
| `twilio_messaging` | Inbound / Outbound | Receive and send SMS/MMS, WhatsApp, and RCS messages. |
| `twilio` | Sync / Outbound (voice) | Answer and place calls from an existing Twilio phone number. |
| `webhook` | Inbound / Outbound / Sync | Generic HTTP. Sync = HTTP request/response; Inbound = fire-and-forget; Outbound = call out to your URL. |
| `websocket` | Sync (real-time chat) | Persistent bidirectional connection; each message is its own run. |
| `whatsapp` | Inbound / Outbound | WhatsApp Cloud API through the customer's Meta app: receive messages to a business number; send text replies or approved templates. |

Common dashboard flow: open the agent's detail page → Connector Flow → **Add inbound connector** (inbound and sync types, including voice and WebSocket) or **Add outbound connector** → **Create New Connector** → pick a type → configure → save. Connector names have 1 to 200 characters. A connector's mode can't change while agents are linked ("Unlink the agents before changing the mode."). A voice connector's mode can't change after creation, even without linked agents. Kafka, RabbitMQ, MQTT, SQS, S3, Postgres, and email connectors reach private endpoints through the **connic bridge** of their Connection: set it on the Connection (dashboard Connection drawer, or the Connection's top-level `bridge` in REST/MCP), never on the connector. An outbound webhook sets `bridge` on the connector. Private MCP servers that an agent consumes use `mcp_servers[].bridge` in agent YAML instead.

At the plan's active connector limit, a new connector is created paused and resuming one returns 403. Telegram, Twilio (voice or messaging), Telnyx (voice or messaging), Linear, Shopify, GitLab, Pipedrive, and WhatsApp connectors change provider settings on create, so at the limit creating one returns 403 instead, and a code deploy that would create or bring back one fails.

File inputs accept up to 50 complete files, with at most 32 MiB combined. Invalid or unavailable optional files are omitted individually. At the first count or size overflow, connic keeps the files already accepted and omits that file and all later files. `file_issues` records omissions; input review can also omit file content that exceeds its budget. Runs can continue with only text and metadata. Under **Linked Agents**, enable **Files required** (link config `files_required: true`) to fail before execution if files are absent or any file is omitted. It's allowed only on types whose events can carry files (`webhook`, `mcp`, `websocket`, `email`, `outlook`, `gmail`, `s3`, `kafka`, `rabbitmq`, `mqtt`, `azure_service_bus`, `sqs`, `telegram`, `twilio_messaging`, `telnyx_messaging`, `whatsapp`) and never on outbound connectors; lint, REST, and MCP reject it elsewhere.

## Outbound connector modes

Twilio Voice, Telnyx Voice, and SIP Voice place calls when runs start; they have no outbound link settings. The delivery modes below apply to other outbound connectors:

- **Automatic outbound connector** — sends once after a completed run. Choose **All runs**, or choose **Only selected inputs** to send only when `run.connector_id` matches one of the selected inbound or sync connector IDs. All runs also includes manual, cron, and `trigger_agent` runs with no inbound source. Existing links without outbound connector settings remain **Automatic / All runs**.
- **Agent-tool outbound connector** — injects an editable connector tool into an LLM agent (other agent types are rejected). Its default `action_name` is `send_to_<normalized connector name>`. Keep it unique across that agent's custom, predefined, connector, and MCP tools. The model decides whether and when to call it and may call several agent-tool outbound connectors in one run.
- **Middleware outbound connector** — project code calls it with `await send_connector(action_name, payload)`, and the model cannot access it.

Source filters and `StopProcessing(..., publish_outbound=False)` apply only to automatic outbound connectors. The flag does not undo an agent-tool or middleware outbound connector call.

Deployment tests do not deliver outbound messages. Automatic outbound connectors are suppressed, while calls to agent-tool and middleware outbound connectors are recorded as mocked tool calls for assertions.

For these delivery modes, each connector owns its payload schema, credentials, destination resolution, wire formatting, retries, and bridge routing. The model and middleware receive no stored secrets or destination values. Agent-tool and middleware outbound connector payloads are:

| Connector | Payload |
| --- | --- |
| HTTP Webhook | `{"payload": { ... }}` |
| Kafka | `{"payload": { ... }, "key": "optional-string"}` |
| SQS | `{"payload": { ... }}` |
| RabbitMQ | `{"payload": { ... }}` |
| MQTT | `{"payload": { ... }, "user_properties"?: {string: string}}` |
| Azure Service Bus | `{"payload": { ... }, "subject"?: string, "correlation_id"?: string, "session_id"?: string, "application_properties"?: {string: string \| number \| boolean}}` |
| Google Pub/Sub | `{"payload": { ... }, "attributes"?: {string: string}, "ordering_key"?: string}` |
| Email, Outlook, Gmail | `{"body": "required", "to"?: string \| string[], "subject"?: string, "html_body"?: string, "cc"?: string \| string[], "bcc"?: string \| string[], "reply_to"?: string}` |
| Telegram | `{"text": "required", "chat_id"?: string \| integer}` |
| Slack | `{"text": "required", "channel_id"?: string, "thread_ts"?: string}` |
| Microsoft Teams | `{"text": "required"}` |
| GitHub | `{"body": "required", "title"?: string, "issue_number"?: integer}` |
| Jira | `{"body": "required", "summary"?: string, "issue_key"?: string}` |
| Linear | `{"body": "required", "title"?: string, "issue_id"?: string}` |
| Notion | `{"body": "required", "title"?: string, "page_id"?: string}` |
| GitLab | `{"body": "required"}` |
| Pipedrive | `{"body": "required", "deal_id"?: integer, "lead_id"?: string, "person_id"?: integer, "org_id"?: integer}` |
| Shopify | `{"tags"?: string[], "remove_tags"?: string[], "note"?: string, "replace_note"?: boolean, "order_id"?: string}`; supply tags, remove_tags, or note |
| Google Sheets | `{"row": {"<column name>": value, ...}}` |
| Telnyx Messaging | `{"text"?: string, "to"?: string, "media_urls"?: string[], "media_type"?: "image" \| "video" \| "document" \| "audio" \| "sticker", "whatsapp_template"?: object}`; supply text, media, or template content |
| Twilio Messaging | `{"text"?: string, "to"?: string, "media_urls"?: string[], "content_sid"?: string, "content_variables"?: object}`; supply text, media, or template content |
| WhatsApp | `{"text"?: string, "to"?: string, "template"?: {"name": string, "language": string, "parameters"?: string[] \| object}}`; supply text or a template |

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
dev:                              # connic dev environments the list doesn't name
  config:
    topic: orders-dev
    group_id: connic-orders-dev
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

- **`type`**: any connector type; `slack`, `teams`, and `github` need a Connection whose setup was finished in the Dashboard. Use `webhook` for HTTP webhooks (`http` also accepted); `twilio`, `telnyx`, and `sip` are the voice connectors. **`mode`** defaults to `sync` for voice connectors. Other types require it when they have more than one mode; otherwise it defaults to the only one.
- **`name`**: optional display name, 1 to 200 characters; defaults to the file name.
- **`config`**: the connector's settings, the same object the REST API and connic MCP take. `connic lint` and the deploy reject unknown fields, fields that belong to the Connection, fields that don't apply to the mode (`require_auth` or `rate_limit` on an outbound webhook, `group_id` on outbound Kafka, `chat_id` on inbound Telegram, S3 `region` without `include_content: true`, WebSocket `progress_events` with `streaming: false`), missing required fields such as cron `schedule`, and values outside the field's type, options, or range. Lint also checks a cron `schedule` has exactly five fields or is a macro (`@hourly`, `@daily`, `@midnight`, `@weekly`, `@monthly`, `@yearly`, `@annually`; six- and seven-field expressions are rejected), `rate_limit.key` whenever `rate_limit.per_key` is set, numeric Telegram IDs (`allowed_user_ids` entries positive, `chat_id` may be negative), and an outbound webhook `outbound_url` without a `bridge`: `https://`, port 443, no credentials in the URL (whether the host is public is checked when the connector is saved at activation). The file is the connector's full desired state: a key removed from `config` goes back to its default or is cleared, and removing `domain`, or `bridge` from an outbound webhook, detaches the connector.
- **`${VAR}`**: same syntax as `mcp_servers`; resolved once at deploy from the target environment's Variables. Allowed in `config`, `connection`, `bridge`, and `domain`. A missing Variable fails the deploy. The deploy converts each value to its field's type (`true`/`false`, whole numbers) and fails on a value that doesn't fit. Quote phone numbers, sender IDs, and Telegram user IDs in an `allowed_user_ids` list when YAML would read them as numbers (lint: "put it in quotes"); a comma-separated `allowed_user_ids` string also works. Telegram `chat_id` may be an unquoted whole number. Secrets the user supplies (Telegram `bot_token`, Stripe `webhook_secret`, and similar) must be `${VAR}` references; literal values fail the deploy.
- **`connection`**: a saved Connection, by name or ID. Required for `kafka`, `rabbitmq`, `mqtt`, `azure_service_bus`, `sqs`, `postgres`, `email`, `outlook`, `gmail`, `google_sheets`, `jira`, `linear`, `notion`, `shopify`, `gitlab`, `pipedrive`, `teams`, `twilio`, `twilio_messaging`, `telnyx`, `telnyx_messaging`, and `whatsapp`; `s3` needs one only with `include_content: true`, and `google_pubsub` only in outbound mode (an inbound one names none). Their credentials never go in the file. **`bridge`**: Bridge name or ID, for an outbound webhook. Kafka, RabbitMQ, MQTT, SQS, S3, Postgres, and email connectors use the Bridge of their Connection; a connector-level `bridge` on them (and S3 `config.endpoint_url`) is deprecated until 2027-04-30, accepted only when it equals the Connection's value (otherwise the deploy fails), and `connic lint` warns about it. Never write it in new files: to use a different Bridge per environment, override `connection` per environment with a Connection that has that Bridge. **`domain`**: hostname or ID of an active custom domain, for types that support custom domains.
- **Generated values** never go in the file, and `connic lint` rejects them: webhook URLs, webhook/WebSocket/MCP secrets, the outbound webhook signing secret, the S3 URL secret, the Telegram secret token, the Google Pub/Sub `push_endpoint`, and SIP `sip_address`, `trunk_username`, and `trunk_password`. connic generates them on the first deploy and keeps them; the dashboard shows them. An omitted Kafka `group_id` is generated; removing `group_id` from the file keeps the consumer group the connector already uses.
- **Voice** (`twilio`, `telnyx`, `sip`): link exactly one agent with `voice_config`, without outbound link settings. For Twilio and Telnyx, set `config.phone_number: "+4930123456"` (E.164, or the number's Twilio SID starting with `PN` or Telnyx ID); the deploy looks an E.164 number up in the Connection. SIP takes the phone system settings directly in `config`, without a Connection. A literal `hold_audio_path` must name an existing MP3 or WAV file in a project directory such as `assets/`.
- **`agents`**: agent names, or objects with `files_required` and `outbound`. At least one link; every agent must exist in `agents/`. `outbound.usage` is `automatic`, `agent_tool`, or `middleware`; `action_name` is required for `agent_tool` and `middleware`; `source` (automatic only) lists keys of other connector files, each an inbound or sync connector declared in the same environment and linked to the same agent. An agent can use an `action_name` on only one connector per environment, and `agent_tool` needs an `llm` agent; lint or the deploy fails otherwise. `files_required: true` fails lint on outbound connectors and on types whose events carry no files. Without `outbound`, the link is automatic for all runs, except outbound voice connectors, which place calls when runs start.
- **`environments`**: keys are environment names, matched exactly. When present, the connector exists only in those environments. Overrides can set `name`, `connection`, `bridge`, `domain`, `agents` (replaces the list), and `config` (merged key by key); `null` removes a `connection`, `bridge`, `domain`, or `config` key set at the top of the file. A key that matches no environment in the project fails the deploy.
- **`dev`**: same fields as an override, for every `connic dev` environment that `environments` doesn't name (quick tests included). With `environments`, those get the connector only when `dev` is set; `dev: {}` keeps the top of the file. Dev environments have their own Variables and quick tests start without any, so override `${VAR}` values here.
- **`apps_only`**: `true` on a sync `webhook` or `websocket` connector (top level only, not per environment) makes it accept only requests connic forwards from the apps that list it in `apps/<key>.yaml`. Its secret and **Require Authentication** no longer apply, and its runs get `context["app"]` and `context["end_user"]`. See [Apps](apps.md#calling-agents-through-connectors).

The REST API (`POST`/`PATCH /v1/projects/{project_id}/connectors`) and connic MCP (`create_connector`, `update_connector`) take the same fields: `type`, `mode`, `name`, `connection`, `bridge`, `domain`, and `config` as an object. Differences: a request names one environment (`environment_id`) instead of `environments`; agents are linked one at a time (`agent_name` on create, the link endpoints, MCP `link_connector`), where the link `config` takes an `agents` entry's `files_required` and `outbound` with `source` as connector IDs; secrets are sent as values (not `${VAR}`) and come back as `********`. Responses return `connection`, `bridge`, and `domain` as IDs; for Kafka, RabbitMQ, MQTT, SQS, S3, Postgres, and email connectors, `bridge` is the Connection's Bridge, and sending it back unchanged is fine. An update changes only the fields it sends (a deploy instead makes the connector match its file): a `config` key sent as `null` goes back to its default or is cleared, `domain: null` removes the custom domain, and `bridge: null` removes an outbound webhook's Bridge. In connic MCP, `config` values sent back as `[redacted]` keep their stored values, and errors carry the reason. Changing `mode` returns 400 while agents are linked. Creating a `twilio` or `telnyx` voice connector requires `agent_name` (REST query parameter, MCP argument) naming an agent deployed with `voice_config`. REST, MCP, and code create `slack`, `teams`, and `github` connectors once their Connection's setup is ready; before that the request or deploy fails. Older field names (`config.mode`, `config.connection_id`, `config.bridge_id`, `domain_id`, flat `rate_limit_*`, `phone_number_sid`/`phone_number_id`, credentials inline in `config`, link `outbound.version` and `source: {mode, connector_ids}`, MCP `config_json`), a connector-level `bridge`/`config.bridge_id` on Kafka, RabbitMQ, MQTT, SQS, S3, Postgres, and email connectors and S3 `config.endpoint_url` (accepted only when they equal the Connection's values, otherwise 400), `config.region` on S3 and SQS Connections (ignored), and creating a connector of a type with several modes without `mode` (falls back to the default mode) still work until 2027-04-30 with `Deprecation`/`Sunset` response headers; never write them in new code.

Lifecycle:

- Every deploy resolves the files for its environment into a snapshot stored with the deployment and logs the planned changes (`create`, `update`, `bring back`, `retire`). Environment names, Connections, Bridges, Variables, phone numbers, `config` values, cron expressions, outbound sources, and action names are checked before the build; errors fail the deploy. Provider setup runs at activation.
- Activating a deployment applies its snapshot: a new deploy, a redeploy of an older commit (Variables resolved fresh), and a rollback to an older deployment (Variables as captured at that deploy). `connic dev` applies the files to its test environment the same way, when the session starts and on every upload that changes them; a change that fails rejects the upload with the reason. Test runs and the deploy gate never change connectors.
- Connectors no longer declared, or every code connector when a deployment has no `connectors/` folder, are **retired**: paused, keeping URL, secrets, links, provider bindings, and run history. Retired connectors don't count against the active connector limit, can't be resumed, and are only deleted by hand. Declaring the same key and type again, or rolling back to a deployment that declared it, brings the connector back with its previous pause state.
- Changing `type` retires the old connector and creates a new one. Renaming the file creates a new connector (new URL and secrets) and retires the old one; change `name:` instead.
- New code connectors start active, including Kafka, SQS, RabbitMQ, MQTT, Azure Service Bus, and Postgres. To hold off consuming in an environment, leave it out of `environments:` until ready. Deploys never change the pause state of existing connectors; pause/resume stays in the dashboard. Over the active connector limit, a new connector stays paused, except Telegram, Twilio, and Telnyx connectors: a deploy that would create or bring back one of those at the limit fails.
- Two active connectors (in any environment) can't use the same messaging sender or Telegram bot, and two sync voice connectors can't use the same phone number; the deploy fails. Outbound voice connectors can reuse a number. A retired connector gives these up when another connector needs them and takes them back if it returns.
- Activation is all or nothing: if a provider call fails, nothing is activated and the previous deployment stays live. **Deploy anyway** on the deployment page redeploys and skips the failing connectors. For a rollback, use **Activate anyway** in the activate dialog (REST: `POST /v1/projects/{project_id}/deployments/{deployment_id}/activate?force=true`). Skipped connectors keep their previous state and are retried on the next activation. The activate dialog lists what a rollback would change.
- A code connector and a dashboard connector can't share a name in one environment; the deploy fails. The dashboard, REST API, and MCP refuse to create or rename a dashboard connector to a name a code connector uses (retired ones included).
- Code connectors show an **In code** badge. Active code connectors cannot be edited, deleted, or linked in the dashboard, and REST/MCP return 409; change the file and deploy instead. Pause/resume stays available.

`connic lint` checks connector files offline: shape, types and modes, `config` fields and literal values, generated fields, literal secrets, `${VAR}` syntax, that linked agents exist, outbound sources and action names, and name clashes between files, and warns about deprecated settings such as a connector-level `bridge` on a Connection type. `connic deploy` runs lint first and uploads `connectors/`.

**Connections** are project-wide credential sets. Manage them under **Project Settings → Connections** (lists the connectors using each Connection per environment), the REST API (`/v1/projects/{project_id}/connections`, permissions `connections.view/create/update/delete`), or connic MCP (`list_connections`, `get_connection`, `create_connection`, `update_connection`, `delete_connection`). A request puts the fields in `config` and the Bridge of a Kafka, RabbitMQ, MQTT, Postgres, email, SQS, or S3 Connection in the top-level `bridge` (name or ID; `null` connects directly). Every create and update is checked: missing required fields and unknown fields return 400, and an update is also checked against every connector that uses the Connection (400 naming the connector that couldn't run with the new values). PATCH merges: omitted keys keep their values, `null` removes an optional field (such as `ssl_ca_pem`), and a secret sent as `********` (MCP: `[redacted]`) or empty keeps its stored value. Changing a host, server address, endpoint URL, or the Bridge requires sending the secrets bound to it again (400 otherwise); the dashboard asks for them. Secrets are write-only and masked on read. Slack, Microsoft Teams, and GitHub Connections can only be created in the dashboard. Rotating a Connection updates every connector that uses it without a deploy; deleting one is blocked while a connector uses it.

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

## outlook

Microsoft 365 mail through Microsoft Graph, for work and school mailboxes (shared mailboxes too, not Outlook.com). Use it instead of `email` for Exchange Online, which no longer accepts IMAP password sign-in. Inbound and outbound are **separate connectors** that share an `outlook` Connection.

- **Connection**: the customer's own single-tenant Entra ID app: `tenant_id` (GUID or `contoso.onmicrosoft.com`), `client_id`, `client_secret`. The app needs the Microsoft Graph application permissions `Mail.ReadWrite` and `Mail.Send` with admin consent. To restrict it to certain mailboxes, assign the roles `Application Mail.ReadWrite` and `Application Mail.Send` with RBAC for Applications in Exchange Online and remove the tenant-wide Entra permissions; otherwise the scope has no effect. Changing `tenant_id` or `client_id` requires sending `client_secret` again. No Bridge.
- **Config**: `mailbox` (UPN, required in both modes), inbound `folder` (default `Inbox`; well-known names like `Sent Items`, a top-level folder name, or a path such as `Inbox/Support`) and `mark_as_read` (default `true`), outbound `default_to`.
- **Inbound**: polls the folder about every 30 seconds with a Graph delta query. It starts with mail that arrives after the first poll; mail already in the folder isn't processed. The payload has the same keys as the `email` connector (`subject`, `from`, `from_address`, `to`, `date`, `message_id`, `body_text`, `body_html`, `has_attachments`, `attachments`, `file_issues`) plus `_outlook` (`connector_id`, `mailbox`, `folder`, `message_id` = the immutable Graph message ID, `timestamp`) instead of `_email`. Attachments follow the email rules; attached Outlook items and cloud links are omitted as unsupported. A rejected sign-in or an unknown mailbox or folder stops polling until the connector or its Connection is saved; missing permissions (403) are retried with backoff.
- **Outbound**: a run started by an inbound Outlook connector on the same mailbox and Connection gets a reply in the thread unless the output sets `to`. Otherwise it sends a new email to `to`, then `default_to` (`missing_recipient` without either). Fields match the Email payload schema; `html_body` replaces `body` (one body), and replies keep the conversation subject. Mail lands in Sent Items. Only unreachable Graph and throttling (429) are retried; a server error or broken connection after hand-off is not, to avoid duplicates. Exchange Online allows 30 messages per minute and 10,000 recipients per day per mailbox.

Full setup: [Outlook documentation](https://connic.co/docs/v1/connectors/outlook).

## gmail

Google Workspace mail through the Gmail API, for Workspace users with Gmail (not groups, not @gmail.com consumer accounts, which use `email` with an app password). Inbound and outbound are **separate connectors** that share a `gmail` Connection.

- **Connection**: type `gmail`, shown as "Google Workspace" in the dashboard; one field, `service_account_key`, the contents of a JSON key file of the customer's own Google service account (stored encrypted; saving checks it's a service account key with a readable private key). The same Connection serves `google_sheets` connectors. In the Google Cloud console: enable the Gmail API (and the Google Sheets API for Sheets), create a service account (no roles), Keys → Add key → Create new key → JSON. Organizations created on or after 2024-05-03 block key creation by default (`iam.disableServiceAccountKeyCreation`); an organization policy administrator has to allow it for the project. Gmail needs domain-wide delegation: a super admin adds the key's numeric `client_id` with the scope `https://www.googleapis.com/auth/gmail.modify` under Admin console → Security → Access and data control → API controls → Manage Domain Wide Delegation. Delegation covers every user of the domain; connic acts only as each connector's `mailbox`. Keys don't expire by default; rotate by pasting a new key into the Connection, then deleting the old one. No Bridge.
- **Config**: `mailbox` (required in both modes), inbound `label` (default `INBOX`; a system label by ID such as `INBOX`, `STARRED`, `IMPORTANT`, or a user label by name, case-insensitive, nested as `Parent/Child`) and `mark_as_read` (default `true`, removes `UNREAD` once every linked agent received the email), outbound `default_to`.
- **Inbound**: follows the label through the Gmail history about every 30 seconds. It starts with mail that arrives after the first poll; mail already there, or an older email labeled later, isn't processed. The payload has the same keys as the `email` connector (`subject`, `from`, `from_address`, `to`, `date` = the Date header or else Gmail's receive time, `message_id` = the Message-ID header, `body_text`, `body_html`, `has_attachments`, `attachments`, `file_issues`) plus `_gmail` (`connector_id`, `mailbox`, `label`, `message_id` = the Gmail message ID, `thread_id`, `timestamp`). Gmail doesn't convert between text and HTML, so one body can be empty. Attachments follow the email rules. If Gmail's history expired (kept about a week, rarely hours), connic rereads the label from the last complete poll minus 5 minutes and skips email agents already received. A rejected sign-in (deleted key, missing delegation, mailbox outside the domain), an unknown label, or a 400/401/403/404 stops polling until the connector or its Connection is saved; rate limits, network and server errors are retried with backoff up to 15 minutes.
- **Outbound**: a run started by an inbound Gmail connector on the same mailbox and Connection gets a reply in the same Gmail thread unless the output sets `to`: to the incoming Reply-To or sender, subject `Re: <subject>` (no second `Re:`), with In-Reply-To/References. Otherwise it sends a new email to `to`, then `default_to` (`missing_recipient` without either), subject `subject` or `Agent Response`. Fields match the Email payload schema; with `body` and `html_body` the email has both parts, with only `html_body` it's HTML only. No attachments. Mail lands in the Sent folder. Only unreachable Gmail and rate limits are retried (3 attempts); a server error or broken connection after hand-off is not, to avoid duplicates. Google Workspace allows 2,000 messages per user per day; the Gmail API allows 6,000 quota units per minute per user and project.

Full setup: [Gmail documentation](https://connic.co/docs/v1/connectors/gmail).

## google sheets

New rows of one sheet in, appended rows out, through the same `gmail` (Google Workspace) Connection and service account as `gmail`. No domain-wide delegation: share the spreadsheet with the key's `client_email` (ends in `.iam.gserviceaccount.com`), Viewer for inbound, Editor for outbound; enable the Google Sheets API in the service account's project. If the Workspace's Drive sharing settings block outside sharing, an admin has to allow that address.

- **Config**: `spreadsheet` (required; URL or ID, stored as the ID) and optional `sheet` (tab name, exact; empty = first sheet).
- **Inbound**: polls about every minute, up to 500 rows per poll. Row 1 holds the column names. The first poll starts after the sheet's last row; existing rows aren't processed, empty rows are skipped, and changing `spreadsheet` or `sheet` starts again. Payload: `{"row": {"<column name>": "<value>", ...}, "row_number": 12, "_google_sheets": {"connector_id", "spreadsheet_id", "sheet", "row_number", "timestamp"}}`. Values are formatted text as the sheet shows them; unnamed or duplicate columns use their column letter (`C`); trailing empty cells are left out. A row is processed the first time it's seen, so rows should be added in one go (form, script, paste); edits to processed rows start nothing. When rows are inserted or deleted above, connic finds the last processed row by its values; if it changed or was deleted, it logs a warning and goes on after the same row number. A rejected sign-in, a spreadsheet not shared with the service account (403), a wrong ID (404), or an unknown sheet stops polling until the connector or Connection is saved.
- **Outbound**: appends one row below the data (inserted, nothing overwritten). Automatic output must be a JSON object whose keys name columns in row 1 (exact match after trimming the column name; column letters for unnamed columns); missing keys leave cells empty, an unknown key fails the delivery naming the keys, and other output fails with "The agent output must be a JSON object whose keys name columns in the sheet's first row". Values are written raw: text starting with `=` stays text, `null` is an empty cell, objects and lists become JSON text. Give the agent an `output_schema` with the columns. Agent-tool/middleware payload: `{"row": {...}}`. Only unreachable Google Sheets and rate limits are retried (3 attempts); a server error or broken connection after sending is not.
- **Quota**: the Sheets API allows 60 reads and 60 writes per minute per user (the service account) and project, 300 per project. Each poll uses two reads, each appended row one read and one write; connectors on the same service account share it.

Full setup: [Google Sheets documentation](https://connic.co/docs/v1/connectors/google-sheets).

## github

Configure GitHub in the Dashboard: create a GitHub Connection (app name, optional organization), click **Create on GitHub** to create a private GitHub App from connic's manifest (GitHub returns to the Connections page and connic stores the app's credentials; finish within an hour), then install the app on the repositories. One Connection is one app and one webhook URL; the app can only be installed on the account or organization that owns it. Then declare the connector in `connectors/<key>.yaml` (`type: github`, `connection: <name>`, `config.repository`, and `config.events` as a list for inbound) or create it in the Dashboard, REST API, or connic MCP; until the Connection's setup is ready, that fails with "Finish the GitHub App setup in the dashboard before using it in a connector".

Each connector names one `repository` (`owner/name`); the app must be installed on it. Inbound connectors list `events`, one per line: `issues`, `issue_comment`, `pull_request`, `pull_request_review`, `pull_request_review_comment`, or `event.action` (such as `issues.opened`) for one action. Several connectors can watch the same repository; an event reaches every one that matches. Events the app causes itself are ignored.

- **Inbound payload**: `event`, `action`, `repository`, `number`, a stable `conversation_id` per issue or pull request, `title`, `text` (body of the issue, pull request, comment, or review), `url`, `sender`, and `raw` (GitHub's full event). Run IDs come from the delivery ID, so a redelivered event starts nothing new. Set `session.key` to `input.conversation_id` when an issue's comments need history; the connector does not enable sessions automatically.
- **Automatic outbound connector**: a run started by an inbound GitHub connector of the same Connection gets the output as a comment on its issue or pull request. Otherwise the output opens an issue in the connector's repository, titled by `title` or the first line. JSON may use `body`, `text`, or `message`, plus `title` and `issue_number` (a comment on that issue or pull request in the connector's repository).
- **Agent-tool / middleware outbound connector**: `body` is required; `title` and `issue_number` are optional, with the same routing.
- GitHub doesn't retry failed deliveries; redeliver them from the app's advanced settings on GitHub within 3 days. GitHub allows about 80 new comments or issues per minute; rate-limited deliveries are retried.

## jira

Jira Cloud only (not Data Center or Server). Inbound and outbound are **separate connectors** that share a `jira` Connection.

- **Connection**: `site_url` (`https://<name>.atlassian.net`), `email` of an Atlassian account, and that account's `api_token`, created without scopes. Connectors act as this account. Atlassian API tokens expire after at most 365 days; replace the token on the Connection before then. Changing `site_url` or `email` requires sending `api_token` again. No Bridge.
- **Config**: inbound takes `events` as a list (`jira:issue_created`, `jira:issue_updated`, `comment_created`) and `jql` (required, such as `project = SUP`); outbound takes `project_key` (required, such as `SUP`) and `issue_type` (default `Task`, as named in the project).
- **Inbound**: saving the connector registers a Jira admin webhook on the site for its events and JQL filter, signed with a generated secret; deleting the connector removes it. The account needs the **Administer Jira** global permission. Each connector owns its own webhook, so several can watch one site. Events the Connection's account causes itself (its own comments and issues) are ignored.
- **Inbound payload**: `event`, `issue_key`, `project`, a stable `conversation_id` per issue (`jira:<site host>/<issue id>`), `summary`, `text` (issue description or comment body), `url`, `author`, `author_account_id`, and `raw` (Jira's full event). Run IDs come from the delivery ID, so a redelivered event starts nothing new. Set `session.key` to `input.conversation_id` when an issue's comments need history; the connector does not enable sessions automatically.
- **Automatic outbound connector**: a run started by an inbound Jira connector of the same Connection gets the output as a comment on its issue. Otherwise the output creates an issue of `issue_type` in `project_key`, summarized by `summary` or the first line. JSON may use `body`, `text`, or `message`, plus `summary` and `issue_key` (a comment on that issue, such as `SUP-12`). Text is posted as plain text paragraphs.
- **Agent-tool / middleware outbound connector**: `body` is required; `summary` and `issue_key` are optional, with the same routing.
- When Jira rate-limits a comment or issue (including its per-issue write limits), connic waits and retries the delivery.

## linear

Inbound and outbound are **separate connectors** that share a `linear` Connection.

- **Connection**: `api_key`, a personal Linear API key (Linear: Settings → Account → Security & Access). Saving the Connection checks the key and stores its member and workspace name. Connectors act as the key's member. No Bridge.
- **Config**: `team` (required, the team key that prefixes its issue IDs, such as `ENG`); inbound also takes `events` as a list: `Issue` or `Comment` for created and updated ones, or `Issue.create`, `Issue.update`, `Comment.create`, `Comment.update` for one action (default `Issue.create` and `Comment.create`).
- **Inbound**: saving the connector registers a webhook for its team in Linear (resource types Issue and/or Comment, as `events` need), signed with a generated secret; changing the team, Connection, or resource types registers a new one, and deleting the connector removes it. Linear only lets workspace admins create webhooks, so inbound needs an admin's key. Each connector owns its own webhook, so several can watch one team. Deliveries older than a minute are rejected. Events the key's member causes (the agent's own comments and issues, and that person's own changes) are ignored, so recommend a Linear member just for the agents. Comments outside an issue start no runs.
- **Inbound payload**: `event` (`Issue` or `Comment`), `action` (`create` or `update`), `issue` (such as `ENG-42`), a stable `conversation_id` per issue (`linear:<issue id>`), `title`, `text` (issue description or comment, Markdown), `url`, `actor`, and `raw` (Linear's full payload). Run IDs come from the `Linear-Delivery` header, so a redelivered event starts nothing new. Set `session.key` to `input.conversation_id` when an issue's comments need history; the connector does not enable sessions automatically.
- **Automatic outbound connector**: a run started by an inbound Linear connector of the same Connection gets the output as a comment on its issue; when a comment started the run, the reply goes into that comment's thread. Otherwise the output creates an issue in `team`, titled by `title` or the first line, with the text as its description. JSON may use `body`, `text`, or `message`, plus `title` and `issue_id` (a comment on that issue, such as `ENG-12`). Text is Markdown.
- **Agent-tool / middleware outbound connector**: `body` is required; `title` and `issue_id` are optional, with the same routing.
- Linear allows 2,500 API requests per hour per member, shared by that member's keys. When Linear rate-limits a comment or issue, connic waits and retries the delivery.

## notion

Inbound and outbound are **separate connectors** that share a `notion` Connection.

- **Connection**: `integration_token`, the secret of a Notion internal integration (notion.so/my-integrations). Saving the Connection checks the secret and stores the integration's bot user and workspace name. The integration only sees pages and databases shared with it (page menu → Connections). Capabilities: Read content for inbound, Read comments for `comment.created`, Insert comments and Insert content for outbound. No Bridge.
- **Config**: inbound takes `events` as a list of `page.created`, `page.properties_updated`, and `comment.created` (default `page.created` and `comment.created`). Both modes take an optional `data_source`, a database or data source link: inbound receives only events of its pages, outbound creates pages there.
- **Inbound**: a Notion webhook subscription belongs to the integration, so the customer creates one per Connection in Notion's Webhooks tab with the Connection's webhook URL (shown in the Connection). Notion then sends a verification token to that URL; connic stores and shows it, and the customer pastes it back under Verify in Notion. Every event is checked with it (`X-Notion-Signature`). Once a signed event arrived, new tokens are ignored unless the Connection's `accept_verification_token` is on. Events the integration causes are ignored. Notion usually sends events within a minute, batches property changes within about a minute, can deliver out of order, and aims for at-most-once delivery, so an event can be missing.
- **Inbound payload**: connic reads the page (and comment) from Notion first: `event`, `page_id`, `title`, `url`, `data_source_id`, `properties`, `text` (page content in Markdown, or the comment's text), `authors`, a stable `conversation_id` per page (`notion:<page id>`), `comment_id` and `discussion_id` for comments, and `raw` (Notion's event). Run IDs come from the event ID, so a resent event starts nothing new. Set `session.key` to `input.conversation_id` when a page's events need history.
- **Automatic outbound connector**: a run started by an inbound Notion connector of the same Connection gets the output as a comment on its page; when a comment started the run, the reply goes into that comment's discussion. Otherwise the output creates a page in `data_source`, titled by `title` or the first line, with the text as Markdown content. JSON may use `body`, `text`, or `message`, plus `title` and `page_id` (a comment on that page, link or ID). Comments are plain text.
- **Agent-tool / middleware outbound connector**: `body` is required; `title` and `page_id` are optional, with the same routing.
- Notion allows an integration 180 requests per minute (600 on Business and Enterprise). Each inbound event takes two. When Notion rate-limits a comment or page, connic waits and retries the delivery.

## whatsapp

Inbound and outbound are **separate connectors** that share a `whatsapp` Connection.

- **Connection**: `app_id`, `app_secret`, `access_token` (a system user token for the app with `business_management`, `whatsapp_business_management`, and `whatsapp_business_messaging`, with the WhatsApp Business Account assigned to the system user), and `waba_id`. connic generates a verify token on the Connection. No Bridge.
- **Config**: `phone_number_id` (required, a number of the Connection's WhatsApp Business Account); outbound also takes `to` (optional default recipient in E.164).
- **Inbound**: saving the connector subscribes the app to the WhatsApp Business Account (only if not yet subscribed) and sets the connector's webhook URL as the phone number's callback URL; deleting the connector removes it. Other numbers of the account are untouched. Saving fails when the number already has its own callback URL (connic doesn't replace it) or belongs to another connic connector; one inbound connector per number. Deliveries are checked with `X-Hub-Signature-256` and the app secret. Reactions and unsupported messages start no runs. Sent, delivered, read, and failed receipts of outgoing messages go to the connector logs and start no runs.
- **Inbound payload**: `message_id` (the `wamid`), `type`, `text` (message text, file caption, or the tapped button or list title), `from` (sender phone number in E.164, `null` when WhatsApp doesn't share it), `user_id` (the sender's business-scoped WhatsApp user ID, always set), `name`, `to` (business number), a stable `conversation_id` per person and number, `files` / `file_issues` for images, audio, video, documents, and stickers, and `raw` (the message). Run IDs come from the `wamid`, so a redelivered message starts nothing new. Set `session.key` to `input.conversation_id` for history; the connector does not enable sessions automatically.
- **Automatic outbound connector**: the recipient is `to` from the output, else the connector's `to`, else the person whose message started the run (when an inbound WhatsApp connector on the same Connection and phone number received it). JSON may use `text`, `body`, `message`, `response`, or `output` for text (up to 4,096 characters), or `template` with `name`, `language` (such as `en_US`), and `parameters` (a list for numbered or an object for named parameters). Text only reaches people within 24 hours of their last message; outside that window, or for people who haven't written, send an approved template. `to` takes a phone number or a WhatsApp user ID.
- **Agent-tool / middleware outbound connector**: `text` or `template`, plus optional `to`, with the same routing.
- When WhatsApp rate-limits a message, connic waits and retries the delivery.

## gitlab

Inbound and outbound are **separate connectors** that share a `gitlab` Connection.

- **Connection**: `base_url` (default `https://gitlab.com`, or a self-managed instance over HTTPS that is reachable from the internet; GitLab 17.4 or later) and `access_token`, a project, group, or personal access token with the `api` scope. Saving the Connection checks the token and stores its user. Connectors act as that user. No Bridge.
- **Config**: inbound only: `project` (required, the path such as `acme/api`, or the numeric ID) and `events` as a list: `issues`, `merge_requests`, or `notes` (comments), optionally with an action: `open`, `close`, `reopen`, `update` (issues and merge requests), `merge`, `approved`, `unapproved`, `approval`, `unapproval` (merge requests), `create`, `update` (notes), such as `merge_requests.merge` (default `issues.open` and `notes.create`). Outbound takes no config.
- **Inbound**: saving the connector adds a webhook to its project in GitLab with a generated secret token and signing token; changing the events updates it, changing the project or moving the Connection to another instance adds a new one, and deleting the connector removes it. Adding webhooks needs the Maintainer role. Each connector owns its own webhook, so several can watch one project. connic checks `X-Gitlab-Token`, and from GitLab 19.0 also the `webhook-signature`. Events the token's user causes (the agent's own comments, and that person's own changes) are ignored, so recommend a project or group access token, which has a user of its own. Comments on commits and snippets, and confidential issues and their comments, start no runs. GitLab turns a webhook off temporarily after 4 failed deliveries in a row and for good after 40.
- **Inbound payload**: `event` (`issues`, `merge_requests`, or `notes`), `action`, `project` (path), `type` (`issue` or `merge_request`, also for comments), `iid`, a stable `conversation_id` per issue or merge request (`gitlab:<host>/<project id>#<iid>` for issues, `!<iid>` for merge requests), `title`, `text` (description or comment, Markdown), `url`, `author` (username), and `raw` (GitLab's full payload). Run IDs come from the `Idempotency-Key` header, so a redelivered event starts nothing new. Set `session.key` to `input.conversation_id` when the comments need history; the connector does not enable sessions automatically.
- **Automatic outbound connector**: a run started by an inbound GitLab connector of the same Connection gets the output as a comment on its issue or merge request. Any other run has nothing to reply to and the delivery fails. JSON may use `body`, `text`, or `message`. Text is Markdown.
- **Agent-tool / middleware outbound connector**: `body` is required, with the same routing.
- When GitLab rate-limits a comment, connic waits and retries the delivery.

## pipedrive

Inbound and outbound are **separate connectors** that share a `pipedrive` Connection.

- **Connection**: `company_domain` (the part before `.pipedrive.com`) and `api_token` of a Pipedrive user (Pipedrive: account name → Company settings → Personal preferences → API). Changing `company_domain` requires sending `api_token` again. Connectors act as the token's user, and webhooks only send records that user can see. Each user has one token per company; a new token replaces the old one. No Bridge.
- **Config**: inbound takes `record_type` (`deal`, `lead`, `person`, `organization`, `activity`, `note`; default `deal`), `action` (`create`, `change`, `delete`, or `any`; default `create`), and `ignore_api_changes` (default `true`). Outbound takes no config.
- **Inbound**: saving the connector registers a Pipedrive webhook (version 2.0) for its record type and action, with HTTP basic auth credentials connic generates, since Pipedrive doesn't sign webhooks. Changing the record type, action, or Connection registers a new one and removes the old; saving again also registers a new one when Pipedrive deleted the old (it deletes webhooks with no successful delivery for 3 days). Deleting the connector removes it. Each connector owns its own webhook; Pipedrive allows 40 per user. With `ignore_api_changes`, events whose `meta.change_source` is `api` start no runs, which covers the agents' own notes.
- **Inbound payload**: `event` (`<record_type>.<action>`), `record_type`, `action`, `record_id` (a UUID for leads), `title` (deal or lead title, person or organization name, activity subject), `url` (deals, leads, people, organizations; empty on delete), `user_id`, `change_source`, a stable `conversation_id` per record (`pipedrive:<company>/<record_type>/<id>`), and `raw` (Pipedrive's full payload with `data`, `previous`, and `meta`). Run IDs come from `meta.id`, so a retried delivery starts nothing new. Set `session.key` to `input.conversation_id` when a record's events need history; the connector does not enable sessions automatically.
- **Automatic outbound connector**: the output becomes a note on the records named by `deal_id`, `lead_id`, `person_id`, or `org_id` in the output; without them, a run started by an inbound Pipedrive connector of the same Connection gets the note on its deal, lead, person, or organization (for an activity or note, the record it belongs to). Otherwise the delivery fails. JSON may use `body`, `text`, or `message`. Text keeps its paragraphs and line breaks.
- **Agent-tool / middleware outbound connector**: `body` is required; `deal_id`, `lead_id`, `person_id`, and `org_id` are optional, with the same routing.
- Pipedrive waits 10 seconds for an answer (connic answers right away), retries after 3, 30, and 150 seconds, and blocks a webhook for 30 minutes after 10 failed first attempts. When Pipedrive rate-limits a note, connic waits and retries the delivery.

## shopify

Inbound and outbound are **separate connectors** that share a `shopify` Connection.

- **Connection**: `shop_domain` (the store's myshopify.com domain, such as `acme.myshopify.com`; the bare store name works too), `client_id`, and `client_secret` of an app the customer creates in the Shopify Dev Dashboard (dev.shopify.com/dashboard): **Apps → Create app → Start from Dev Dashboard**, name it; on the **Versions** page keep the default App URL, set Webhooks API version `2026-10`, add the scopes `read_orders` for inbound and `write_orders` for outbound (a write scope includes reading), and **Release**; under the app's **Settings → Access requests → Protected customer data → Manage**, select Customer data, plus the name, email, address, or phone fields the agents need (order webhooks carry customer data); **Install app** on the store; copy Client ID and Client secret from the app's **Settings**. The app and the store must belong to the same Shopify organization. Saving the Connection asks Shopify for a token, so wrong values or an app outside the store's organization fail on save. connic gets short-lived tokens from the client ID and secret whenever it calls Shopify; there is nothing else to rotate. Changing `client_id` requires sending `client_secret` again. No Bridge.
- **Config**: inbound only: `topics` as a list from `ORDERS_CREATE`, `ORDERS_PAID`, `ORDERS_FULFILLED`, `ORDERS_PARTIALLY_FULFILLED`, `ORDERS_CANCELLED` (REST names such as `orders/create` work too; default `ORDERS_CREATE` and `ORDERS_PAID`). Outbound takes no config.
- **Inbound**: saving the connector subscribes the app to its topics in the store for the connector's URL; changing the topics updates the subscriptions, and deleting the connector removes them. Shopify retries failed deliveries over about 4 hours and deletes a subscription after 8 failed deliveries in a row; saving the connector again subscribes it again. Every delivery's `X-Shopify-Hmac-Sha256` signature is checked with the client secret. Saving fails when the app lacks the scope (`read_orders` inbound, `write_orders` outbound); adding a scope needs a new released app version that is approved in the store.
- **Inbound payload**: `topic` (such as `orders/create`), `order` (the order name, such as `#1001`), `order_id` (numeric), a stable `conversation_id` per order (`shopify:<shop domain>/<order id>`), `url` (the order in the Shopify admin), and `raw` (the whole order as Shopify sends it, in the app's webhook API version). Run IDs come from the `X-Shopify-Webhook-Id` header, so a redelivered webhook starts nothing new. Set `session.key` to `input.conversation_id` when an order's events need history; the connector does not enable sessions automatically.
- **Automatic outbound connector**: changes the order named by `order_id` in the output (numeric, such as `5841234567`, or `gid://shopify/Order/5841234567`); without it, the order whose webhook started the run, when an inbound Shopify connector of the same Connection started it; otherwise the delivery fails. JSON may use `tags` (list or comma-separated string: tags to add), `remove_tags` (list: tags to remove), `note` (added as a new line to the order note), and `replace_note: true` (replace the note with `note` instead). Plain-text output is added to the note. Output with no tags, remove_tags, or note fails. Changes run in this order: remove tags, add tags, note. The delivery output holds `order_id`, `url`, and `tags_added`, `tags_removed`, and `note` (`appended` or `replaced`) for what changed.
- **Agent-tool / middleware outbound connector**: the same fields (`tags` and `remove_tags` as lists, `order_id` as a string); supply at least one of `tags`, `remove_tags`, or `note`. Same routing.
- When Shopify rate-limits a change, connic waits and retries the delivery.

## kafka

Inbound (Consumer) and Outbound (Producer) modes.

- **Connection**: a project-wide Kafka Connection holds the bootstrap servers, security protocol, SASL credentials, TLS PEMs, and the optional Bridge. The topic and, for inbound, `group_id` and `auto_offset_reset` go in the connector's `config`. Direct connections require bootstrap and discovered brokers to resolve only to public IP addresses; private brokers require a Bridge on the Connection.
- **Inbound payload**: the parsed message value. Metadata is exposed at `_kafka` inside the payload: `{topic, partition, offset, timestamp, key}` — not in `context`. JSON-object values are dispatched with their top-level fields plus `_kafka`; anything else is wrapped under a `message` key — non-JSON values as `{"message": "<raw text>", "_kafka": ...}`, null values (compaction tombstones) as `{"message": null, "_kafka": ...}`. Tombstones DO trigger runs; use `_kafka.key` to identify the deleted entity.
- **Inbound delivery**: the offset is committed once a queued run is stored for every linked agent, not when the runs finish. If storing fails, the offset stays uncommitted and the same message is redelivered after a short delay, holding back later messages on that connector. Values or keys that are not valid UTF-8 are recorded as failed connector runs and committed. A run that fails later is not redelivered and there is no dead-letter topic; the failed run keeps its input under Runs for re-run. A redelivered message (partial dispatch failure, restart before commit, rebalance) reuses its stored run: each agent gets one run per message and consumer group, also across connectors sharing a Group ID. There is no batch mode.
- **Inbound ordering and parallelism**: a connector reads its partitions one message at a time and stores runs in offset order, but runs execute in parallel up to `max_concurrent_runs`. For per-key ordering, set `concurrency: {key: _kafka.key, on_conflict: queue}` on the agent: same-key runs start one at a time in arrival order, a failed run releases its key, and keyless messages are not serialized. For parallel intake on a high-volume topic, create several inbound connectors with the same Group ID linked to the same agent; Kafka splits the partitions between them.
- **Automatic outbound connector**: publishes a run envelope with `run_id`, `agent_name`, `status`, `output`, `error`, `started_at`, `ended_at`, and `token_usage`. If the run was triggered by inbound Kafka, its key is preserved for partition ordering; otherwise the Kafka key falls back to `run_id`. An agent-tool or middleware outbound connector publishes its `payload` as the message value and may set `key` explicitly.
- **Automatic outbound connector gating**: only runs that end `completed` are delivered — failed and cancelled runs are skipped. A run ended via `StopProcessing` counts as completed and is published (the stop message becomes `output`) unless raised with `publish_outbound=False` — see [StopProcessing](tools-and-python.md). Returning `None` or empty output does not suppress the automatic outbound connector.

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

## rabbitmq

Inbound (Consumer) and Outbound (Publisher) modes over AMQP 0-9-1.

- **Connection**: a project-wide RabbitMQ Connection holds `host`, optional `port` (5671 with TLS, 5672 without), `vhost` (default `/`), `username`, `password`, `use_tls` (default true; checks the certificate and hostname), an optional `ssl_ca_pem` for a private CA, and the optional Bridge. Without a Bridge the host must resolve only to public IP addresses. connic never declares queues or exchanges; they must exist.
- **Config**: inbound takes `queue` and `prefetch` (1–1000, default 10); outbound takes `exchange` (empty = default exchange) and `routing_key` (required with the default exchange).
- **Inbound payload**: a JSON object keeps its top-level fields; any other body is wrapped under `message` (empty body = `null`). `_rabbitmq: {queue, exchange, routing_key, message_id, correlation_id, content_type, headers, timestamp, redelivered}` is added; unset properties are `null`, byte header values arrive as text.
- **Inbound delivery**: the message is acked once a queued run is stored for every linked agent, not when runs finish, so long runs never hit RabbitMQ's consumer timeout. If storing fails, connic closes the connection and RabbitMQ requeues the message (and other unacked prefetched ones); the consumer reconnects with backoff (5 s up to 2 min). Each requeue counts as a delivery: quorum queues dead-letter or drop a message after 20 deliveries by default. Bodies that are not valid UTF-8 are rejected without requeue (dead-lettered if the queue has a DLX) and recorded as failed connector runs. A redelivered message reuses its stored run: the run ID comes from `message_id` within queue and vhost, or from a hash of body and headers when `message_id` is unset, so identical messages without `message_id` start one run per agent. A run that fails later is not redelivered. Connectors on the same queue compete: each message goes to one of them. A refused login or permission stops the consumer until the connector or Connection changes; a missing queue is retried with backoff.
- **Inbound ordering**: messages are dispatched one at a time in arrival order, but runs execute in parallel up to `max_concurrent_runs`. For ordering by a field, set `concurrency: {key: <field>, on_conflict: queue}` on the agent.
- **Automatic outbound connector**: publishes the same run envelope as Kafka and SQS as a JSON body with `content_type: application/json`, persistent delivery mode, `message_id` = run ID, publisher confirms, and `mandatory` (a message no queue takes fails the delivery). An agent-tool or middleware outbound connector publishes its `payload` (its `message_id` is the payload's `run_id`, else a delivery ID stable across retries). Connection errors and unconfirmed publishes get up to 3 attempts; refused logins, missing exchanges and unroutable messages fail at once.

## mqtt

Inbound (Subscriber) and Outbound (Publisher) modes. MQTT 5 only.

- **Connection**: a project-wide MQTT Connection holds `host`, optional `port` (8883 with TLS, 1883 without), optional `username` and `password`, `use_tls` (default true; checks the certificate and hostname), an optional `ssl_ca_pem` for a private CA, an optional `ssl_cert_pem` and unencrypted `ssl_key_pem` for mutual TLS (set together; AWS IoT Core uses these with no username), and the optional Bridge. Without a Bridge the host must resolve only to public IP addresses.
- **Config**: `client_id` (optional; empty generates `connic-` plus 16 hex characters, kept on update) and `qos` (0 or 1, default 1) in both modes. Inbound takes `topic_filter` (`+` and `#` wildcards, each filling a whole level; `$share/` is rejected) and an optional `message_id_field`. Outbound takes `topic` (no wildcards) and `retain` (default false).
- **Client ID**: the subscriber connects with `client_id` and the broker keeps its session under it; each publish connects as `<client_id>-<8 hex>`, so parallel publishes never take over each other's session. Two clients with the same client ID disconnect each other: give every connector its own. Brokers that restrict client IDs (AWS IoT Core policies) must allow `client/<client_id>` for a subscriber and `client/<client_id>-*` for a publisher.
- **Inbound session**: Clean Start 0 with a requested session expiry of one day (brokers may grant less: AWS IoT Core 1 hour by default, Azure Event Grid 1 hour by default). The broker holds QoS 1 messages while connic is disconnected or the connector is paused, until the session expires; QoS 0 messages from that time are lost. Retained messages arrive only when the subscription is new. After a `topic_filter` change the old subscription stays in the session until it expires, and its messages are acknowledged without runs; changing `client_id` too starts a new session.
- **Inbound payload**: a JSON object keeps its top-level fields; any other UTF-8 payload is wrapped under `message` (empty payload = `null`). `_mqtt: {topic, qos, retain, duplicate, user_properties, content_type}` is added; `user_properties` is an object, `content_type` is `null` when unset. Payloads that aren't valid UTF-8 are acknowledged and recorded as failed connector runs.
- **Inbound delivery**: a QoS 1 message is acknowledged (PUBACK) once a queued run is stored for every linked agent; connic only sends successful PUBACKs. If storing fails, connic reconnects without acknowledging it and the broker resends it (backoff 5 s up to 2 min). With `message_id_field`, the run ID comes from that MQTT 5 user property, else that top-level JSON key, so a message whose ID already started a run on the same topic never starts another, even when the broker resends it under a new packet ID (AWS IoT Core does). Without it, connic remembers unacknowledged QoS 1 messages and a resend in the same session reuses their runs; identical messages always run; only a subscriber restart between storing and acknowledging can start a second run. When QoS 0 messages arrive faster than runs start, connic drops further QoS 0 messages once 100 are waiting. Every connector gets every matching message (no shared subscriptions). A refused login, certificate, client ID, or subscription stops the subscriber until the connector or Connection changes.
- **Inbound ordering**: messages are dispatched one at a time in arrival order, but runs execute in parallel up to `max_concurrent_runs`. For ordering by a field, set `concurrency: {key: <field>, on_conflict: queue}` on the agent.
- **Automatic outbound connector**: publishes the same run envelope as Kafka and SQS as a JSON payload with the connector's `qos` and `retain`, content type `application/json`, and a `message_id` user property = run ID; with QoS 1 it waits for the PUBACK. An agent-tool or middleware outbound connector publishes its `payload` and adds its optional `user_properties` (its `message_id` is the payload's `run_id`, else a delivery ID stable across retries). Connection errors and unconfirmed publishes get up to 3 attempts; refused logins and PUBACK errors (such as not authorized) fail at once.
- **Broker limits**: AWS IoT Core supports QoS 0 and 1 only and 128 KB payloads. Azure Event Grid closes the connection on a non-success PUBACK, supports QoS 0 and 1 only and 1 MB messages, and ends a session that holds more than 100 messages or 1 MB while connic is disconnected.

## azure_service_bus

Inbound (Receiver) and Outbound (Sender) modes. No Bridge: Service Bus is reached over the internet.

- **Connection**: a project-wide Azure Service Bus Connection holds `namespace` (host name such as `contoso.servicebus.windows.net`; public Azure only) and `auth_method`: `sas` (default) with `sas_key_name` and `sas_key` of a shared access policy (Listen for inbound, Send for outbound), or `entra` with `tenant_id`, `client_id`, and `client_secret` of a Microsoft Entra ID app holding the Azure Service Bus Data Receiver (inbound) or Data Sender (outbound) role. connic never creates entities; they must exist.
- **Config**: `entity_type` (`queue` default, or `topic`), then `queue`, or `topic` plus `subscription` (inbound only).
- **Inbound payload**: a JSON object keeps its top-level fields; any other body is wrapped under `message` (empty body = `null`). `_azure_service_bus: {namespace, queue | topic + subscription, message_id, sequence_number, correlation_id, subject, content_type, reply_to, application_properties, enqueued_time, delivery_count}` is added; unset properties are `null`.
- **Inbound delivery**: peek-lock, one message at a time in arrival order. connic renews the lock while it stores a queued run for every linked agent, then completes the message, before runs execute. If storing fails, connic abandons the message and reconnects with backoff (5 s up to 2 min); after the entity's max delivery count (default 10) Service Bus dead-letters it. Bodies that are not valid UTF-8 are dead-lettered with reason `Unparsable body` and recorded as failed connector runs. The run ID comes from `message_id` within the entity, or the sequence number when unset, so a redelivered message reuses its stored run. Queues and subscriptions that require sessions aren't supported: the receiver stops with an error. A refused sign-in, missing right or role, or a missing or disabled entity also stops it until the connector or Connection changes.
- **Inbound ordering**: runs execute in parallel up to `max_concurrent_runs`. For ordering by a field, set `concurrency: {key: <field>, on_conflict: queue}` on the agent.
- **Automatic outbound connector**: sends the same run envelope as Kafka and SQS as a JSON body with `content_type: application/json` and `message_id` = run ID, without a session ID. An agent-tool or middleware outbound connector sends its `payload` and can set `subject`, `correlation_id`, `session_id` (required for entities that require sessions), and `application_properties`. Network and service errors get up to 3 attempts; a refused sign-in, missing right or entity, and rejected messages fail at once.

## google_pubsub

Google Cloud Pub/Sub: Inbound (Push subscription) and Outbound (Publish) modes, as **separate connectors**. No Bridge, no custom domain. Every linked agent gets a run per message.

- **Connection**: only outbound connectors use one (required there); an inbound connector has none, so omit `connection` in YAML. Type `google_pubsub`, "Google Pub/Sub" on the Connections page, one field `service_account_key`: the contents of a JSON key file of the customer's service account, which needs `roles/pubsub.publisher` on the topic (`gcloud pubsub topics add-iam-policy-binding TOPIC_ID --member="serviceAccount:SERVICE_ACCOUNT_EMAIL" --role="roles/pubsub.publisher"`; key: `gcloud iam service-accounts keys create key.json --iam-account=SERVICE_ACCOUNT_EMAIL`). Saving checks it's a service account key with a readable private key. Organizations created on or after 2024-05-03 enforce `constraints/iam.managed.disableServiceAccountKeyCreation` by default; an organization policy administrator has to allow key creation for the project.
- **Inbound config**: `push_service_account` (required; email of the service account the push subscription authenticates with) and optional `audience` (leave empty when the subscription uses its default audience, the push endpoint). Generated: `push_endpoint` = `https://connect.connic.co/google-pubsub/<connector-id>` ("Push endpoint" in the details drawer).
- **Inbound setup**: the push auth service account must be in the subscription's project, and the principal creating the subscription needs `iam.serviceAccounts.actAs` on it (such as `roles/iam.serviceAccountUser`). Google's docs grant `roles/iam.serviceAccountTokenCreator` to the Pub/Sub service agent `service-PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com`. Create the subscription with `gcloud pubsub subscriptions create SUBSCRIPTION_ID --topic=TOPIC_ID --push-endpoint=PUSH_ENDPOINT --push-auth-service-account=SERVICE_ACCOUNT_EMAIL` (optional `--push-auth-token-audience=AUDIENCE`, then set the same `audience`); for an existing one use `modify-push-config` with the same flags. Keep the default wrapped payload: `--push-no-wrapper` subscriptions get 400. A code connector's push endpoint exists after its first deploy.
- **Push authentication**: every request needs Google's OIDC token (`Authorization: Bearer`). connic checks Google's signature, issuer `accounts.google.com`, the audience (`audience`, else the push endpoint), that the token's email is `push_service_account`, and that it's verified. Otherwise 401, and Pub/Sub retries.
- **Inbound payload**: the base64 data is decoded. A JSON object keeps its keys at the top level; any other data (text, array, number) goes under `message`; a message with only attributes has `message: null`. Data that is neither JSON nor UTF-8 text gets 400. `_google_pubsub: {subscription, message_id, publish_time, ordering_key (null when none), attributes}` is added. No files.
- **Inbound delivery**: 200 once a queued run is stored for every linked agent (Pub/Sub acknowledges). 503 when the Project has no active deployment or runs couldn't be stored, 429 when the connector's request limit is used up; Pub/Sub retries every non-200 with exponential backoff (100 ms to 60 s) until the message expires or the subscription's dead-letter topic takes it. Requests over 10 MiB get 413, so data above roughly 7.5 MB (base64) can't be received. Run IDs come from the message ID, so a redelivered message starts no new run. Runs execute in parallel; for per-key order, set `concurrency: {key: _google_pubsub.ordering_key, on_conflict: queue}` on the agent.
- **Automatic outbound connector**: one message per run; data is the same JSON run envelope as Kafka and SQS, with attributes `run_id`, `agent_name`, `status`. A run started from an inbound Pub/Sub message with an ordering key keeps that ordering key. An agent-tool or middleware outbound connector publishes its `payload` as JSON data with optional `attributes` (text values only; other values fail the delivery) and `ordering_key`. The connector run shows the published message ID. Rate limits, Google server errors, and connection errors get up to 3 attempts; others fail at once (403: grant the Publisher role; 404: check the topic). At-least-once: a retry after a lost answer can publish twice, so dedupe automatic messages by the `run_id` attribute. Same-ordering-key messages arrive in order only on subscriptions with message ordering on. Google limits: 10 MB data, 100 attributes (keys 256 bytes, values 1024 bytes), 1 MBps per ordering key.

Full setup: [Google Pub/Sub documentation](https://connic.co/docs/v1/connectors/google-pubsub).

## s3

**Inbound only**, driven by S3 object events. Wire S3 → SNS HTTP subscription → connector URL, **or** S3 → EventBridge → connector URL.

- **Config**: bucket, event mode (**Object Created** by default, or **All Events** to include deletes/restores), optional **prefix/suffix filters**, optional `include_content`, `region` (the bucket's AWS region; required with `include_content: true` and rejected without it in files), and max file size (an integer from 1 to 32 MiB, default 10). Each object event produces one input; content downloads use the configured limit, and omitted content arrives as event metadata with `content.skipped: true` and reasons in `file_issues`.
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
- **Inbound payload**: includes `text`, `from`, `to`, `channel`, `message_id`, a stable `conversation_id`, and the original Telnyx message payload under `raw`. Downloaded media appears in `files`, with up to 50 attachments and a combined 32 MiB limit. Set `session.key` to `input.conversation_id` when conversations need history; the connector does not enable sessions automatically.
- **Outbound content**: send text, public media URLs, or an approved WhatsApp template. An explicit `to` wins, followed by the configured default recipient and then trusted matching inbound context. A successful connector run records Telnyx's API acceptance and message ID; check Telnyx for final delivery.

Full setup: [Telnyx Messaging documentation](https://connic.co/docs/v1/connectors/telnyx-messaging).

## Voice call settings

Twilio Voice, Telnyx Voice, and SIP Voice use `sync` for incoming calls and `outbound` for outgoing calls. Each connector links to exactly one deployed agent with `voice_config`; its mode can't change after creation.

In outbound mode, every run of the agent places a call. Link only one enabled outbound voice connector to the agent. Set the E.164 number in the run input's `to` field, or set `config.to` as the default. Start runs from an inbound connector or `trigger_agent`, for example. Runs beyond `max_concurrent_runs` wait before dialing. `config.ring_seconds` allows 5–120 seconds (default 30); unanswered or unconnected calls fail the run with a reason. The agent receives the run input, so include the context it needs for the conversation; context has no call ID or phone numbers, and `connector_id` is the connector that started the run. With a `greeting` the agent speaks first when someone answers; without one it waits for them to speak. The agent gets through phone menus with the built-in `press_keys` tool and can wait in a queue with `wait_on_hold` (see agent-yaml.md).

In sync mode, callers beyond the agent's `max_concurrent_runs` wait in order for a free slot. `config.max_wait_seconds` allows 0–900 seconds (default 30); after that, callers go to `config.fallback_number`, or the call ends and its run fails. 0 sends them to fallback right away, or ends the call without one. While waiting, callers hear a ringing tone or the MP3/WAV file named by `config.hold_audio_path`, played on repeat. The file deploys with the project and may be up to 10 minutes long; missing or unplayable files use the ringing tone and produce a connector log entry.

All three support `config.transfer_numbers`: up to 20 E.164 numbers that tools may pass to `TransferCall`. If a transfer number doesn't answer, the call ends. Sync connectors also forward accepted calls the agent can't finish to the fallback number, or end the call without one; the fallback number never covers transfers or outgoing calls. Calls connic doesn't answer, such as while the connector is paused, are not forwarded.

Sync connectors take `config.allowed_callers`: up to 100 E.164 numbers, as a list or one string with line breaks or commas. When set, connic rejects calls from other and hidden numbers before answering (no run starts) and writes each rejection with the number to the connector logs. SIP also matches allowed numbers sent without the +, with a 00 prefix, or in national format. Caller numbers can be faked, so the list is a filter, not authentication. `connic lint` checks the numbers.

Data location on the phone side: a Twilio Connection's region only selects the Twilio API connic uses, and the number's settings in Twilio decide which region handles calls; Telnyx routes audio through the AnchorSite of the connector's Call Control application, which customers set in the Telnyx portal and connic leaves unchanged (a new application for a new number starts with Telnyx's default); SIP calls follow the phone system's route.

## telnyx voice

Telnyx Voice answers and places calls from an existing Telnyx number. In sync mode, it connects incoming calls to the linked voice agent. Save the account API key and base64-encoded Ed25519 public key (copy it from the Mission Control Portal; connic can't verify it on save, and signature failures appear in the connector logs), select a voice-capable number (numbers the connector can't use are shown disabled with the reason), and link exactly one deployed voice agent. The agent is required when the connector is created: the dashboard asks for it, REST takes the `agent_name` query parameter, and MCP `create_connector` takes `agent_name`. In sync mode, connic creates a dedicated Call Control application and replaces the number's incoming-call routing, and clears a technical prefix or destination rewrite on the number. The previous values go to the connector logs and the project audit log; removing the connector does not restore them.

Call details: `call_id` is the Telnyx call control ID, `caller_name` comes from `caller_id_name`, and `raw` is the `call.initiated` payload including `custom_headers` and `sip_headers`. Forwarding arrives as a `Diversion` entry in `sip_headers`, so `forwarded_from` is not set.

In outbound mode, `config.outbound_voice_profile_id` is required. connic creates a dedicated Call Control application with that profile and leaves the number's incoming-call routing unchanged. Deleting the connector deletes its application.

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
- **Inbound payload**: `text`, `from`, `to`, `channel`, `message_sid`, and a stable `conversation_id`. Downloaded media uses standard connic `files` input, limited to 32 MiB total per message.
- **Conversation history**: explicitly configure the agent with `session: {key: input.conversation_id, ttl: 86400}` and redeploy it. The connector does not enable sessions automatically.
- **Outbound content**: automatic delivery accepts a plain string or the structured payload above. Agent-tool and middleware delivery uses the structured payload. Text has a 1,600-character limit; media URLs must be public HTTP/HTTPS URLs without embedded credentials and satisfy the selected channel's media limits.
- **Recipient precedence**: explicit payload `to`, configured default `to`, then the customer from trusted matching inbound context. Reply fallback requires the same saved connection, account, region, channel, and Twilio sender. RCS recipient addresses use `rcs:+E164`, without automatic SMS fallback.
- **WhatsApp templates**: outside the 24-hour customer service window, use an approved `content_sid` (`HX…`) with `content_variables`. Explicit template content cannot be combined with `text` or `media_urls`. A configured Content SID ignores plain generated text; send variables to fill the template.
- **Delivery status**: a completed outbound connector run means Twilio accepted the API request. Check Twilio Messaging Logs for final delivery status.

Full setup: [Twilio Messaging documentation](https://connic.co/docs/v1/connectors/twilio-messaging).

## twilio voice

Twilio Voice is in beta. It answers and places calls from an existing customer-owned Twilio number. For incoming calls, use sync mode:

1. Deploy a voice agent using a supported realtime model.
2. In Twilio, open a voice-capable number and confirm its incoming-call region. Remove the number from a SIP trunk if it is on one (Twilio has no API to do this through the number). connic replaces an existing incoming-call webhook or TwiML App and writes the previous values to the connector logs and the project audit log.
3. In connic, create a Twilio connection with the Account SID, region, and that region's Auth Token. Supported regions are US1, IE1, and AU1; US1 is the default. Messaging on the same Connection works in US1 (all channels) and IE1 (SMS text only), not AU1.
4. Add the Twilio Voice connector to the agent, reuse the saved connection, and select the number (numbers the connector can't use are shown disabled with the reason). connic links the agent and installs the number's incoming-call webhook with HTTP POST. The voice agent is required at creation: REST takes the `agent_name` query parameter and MCP `create_connector` takes `agent_name`.

The Auth Token must match the selected region; an API key SID and secret are not accepted in its place. One connection can be reused for other numbers in the same account and region. Changing numbers installs the new webhook and removes connic's webhook from the previous number. Deleting the connector removes the webhook only while it still points to connic.

Call details: `call_id` is the Twilio CallSid; `forwarded_from` comes from `ForwardedFrom`, which Twilio sends only when the caller's carrier passes it; `caller_name` requires caller ID lookup on the number; `raw` holds every parameter of Twilio's incoming-call request (location fields, `StirVerstat`, and so on, where Twilio provides them).

In outbound mode, connic leaves the number's incoming-call settings unchanged. Use the same saved Connection and number, and configure the [voice call settings](#voice-call-settings).

connic does not change the number's **Primary handler fails** URL. Twilio uses it when connic rejects a call, for example while the connector is paused or the environment has no active deployment; point it at TwiML that dials a staff number to forward those calls.

Run details show the conversation transcript when transcription is available, tool calls, timings, and errors. Voice configuration and limitations are in [agent-yaml.md](agent-yaml.md#voice-llm-agent).

## sip voice

SIP Voice answers and places calls through a SIP phone system or provider. It uses no saved Connection. `config.setup: register` uses the phone system's `server`, `username`, and `password`; `auth_username` is optional when authentication uses a different username. For incoming calls, connic signs in like a phone. For outgoing calls, it authenticates each call without keeping the line signed in. The password must use ASCII without spaces or semicolons. Choose `config.setup: trunk` for the phone system to send calls to connic using the generated SIP address, username, and password shown on the connector. These credentials stay the same when the connector is edited. `server` is required for outgoing calls with either setup.

`transport` is `tls` (default), `tcp`, or `udp`; `port` defaults to 5061 for TLS and 5060 otherwise. `media_encryption` is `srtp` (default) or `none`. `codecs` is `g722,alaw,ulaw` (default, G.722 first) or `alaw,ulaw` (G.711 only). Optional `caller_id` sets the number or extension outgoing calls come from. Transfers require the phone system to allow them.

```yaml
# connectors/phone-line.yaml
type: sip
mode: sync
config:
  setup: register
  server: sip.example.com
  username: "200"
  password: ${SIP_PASSWORD}
  max_wait_seconds: 60
  hold_audio_path: assets/hold.mp3
agents:
  - phone-assistant
```

Incoming call input has `call_id`, `from`, `to`, and `provider: sip`. Outgoing calls use the run input. Both follow the [voice call settings](#voice-call-settings).

Full setup: [SIP Voice documentation](https://connic.co/docs/v1/connectors/sip-voice).

## slack

Inbound Slack connectors trigger linked agents from signed `app_mention` events. Origin-thread replies require the inbound and outbound connectors to use the same verified Slack Connection.

Configure Slack in the Dashboard: create a Slack Connection, open the generated app manifest, save the Signing Secret, verify the Events request URL, install the app, and save its `xoxb-` Bot User OAuth Token. Then create the inbound or outbound connector and invite the bot to every channel where it should receive mentions or post messages. Declare the connector in `connectors/<key>.yaml` (`type: slack`, `connection: <name>`, optional `config.channel_id` for outbound) or create it in the Dashboard, REST API, or connic MCP once the Slack Connection's setup is ready; before that, it fails with "Complete Slack connection setup in the dashboard before using it in a connector".

One Slack Connection can have one inbound connector across all environments. Link additional agents to that connector; outbound connectors can reuse the Connection. To receive mentions in another environment, create a second Slack app (a new Slack Connection). While a Slack Connection isn't ready (after its signing secret was replaced and the Events URL isn't verified again, or after the app was uninstalled), automatic outbound deliveries are recorded as failed connector runs with `slack_setup_not_ready`.

- **Inbound payload**: includes `text`, `user_id`, `team_id`, `channel_id`, `thread_ts`, and `event_id`.
- **Automatic outbound connector**: a plain final response is sent as text; JSON may use `text`, `message`, or `body` plus optional `channel_id` and `thread_ts`.
- **Agent-tool / middleware outbound connector**: `text` is required; `channel_id` and `thread_ts` are optional. An explicit route wins, then trusted origin context from an inbound connector using the same Slack Connection, then the configured default channel.
- Connection credentials, origin routing, rate-limit handling, and retries stay connector-owned.

## microsoft teams

Type `teams`. Works with the customer's own single-tenant Azure Bot (Microsoft no longer allows new multi-tenant bots). Inbound and outbound are **separate connectors** that share a `teams` Connection. No sync mode: Teams expects its HTTP answer within seconds, so connic acknowledges every message right away and the agent's answer goes out as a follow-up message through an outbound Teams connector.

- **Setup**: in the Azure portal, create an Azure Bot with type of app Single Tenant and a new Microsoft App ID; copy the Microsoft App ID and App Tenant ID from its Configuration page, create a client secret under the app registration's Certificates & secrets (Manage Password), and add the Microsoft Teams channel. In the Dashboard (Project Settings → Connections → New connection → Microsoft Teams), the drawer has three steps: **Bot** (connic signs in as the bot to check the values before saving), **Messaging endpoint** (paste `https://connect.connic.co/teams/<connection id>` into the bot's Configuration page), and **Teams app** (**Download app package**: a zip with `manifest.json` and icons; upload it in the Teams admin center under Teams apps → Manage apps → Upload new app, or in Teams under Apps → Manage your apps → Upload an app → Upload a custom app when custom app upload is allowed; then add the app to the teams, group chats, or people that should use it). The drawer lists the teams once Teams reports the install. Changing the bot name, Microsoft App ID, or the all-messages option gives a new package version to upload as an update of the same app.
- **Connection**: `app_name` (bot name shown in Teams, at most 30 characters), `app_id`, `tenant_id` (the App Tenant ID), `client_secret`, optional `receive_all_channel_messages`. The Connection is set up in the Dashboard only; REST and connic MCP refuse to create it. Connectors in code, REST, and connic MCP use it by name or ID and fail with "Finish the Microsoft Teams setup in the dashboard first" until it is set up. The client secret expires; update the Connection with a new one. No Bridge.
- **Config**: inbound has none; outbound takes optional `channel`, a link from "Get link to channel" in Teams or a channel ID such as `19:…@thread.tacv2`. Declare in `connectors/<key>.yaml` with `type: teams`, `mode`, and `connection: <name>`.
- **Inbound**: every message in a 1:1 chat with the bot, and messages that mention the bot in channels and group chats. With `receive_all_channel_messages`, the app asks for the resource-specific permission to read a team's channel messages (a team owner consents when adding the app), and every channel message arrives. Every inbound connector on the Connection receives every message: link more agents to one connector, and use separate bots for separate environments. Messages from other tenants are ignored; a message Teams delivers twice starts no second run.
- **Inbound payload**: `text` (mention of the bot removed), `conversation_id` (`teams:<Teams conversation ID>`, stable per chat and per channel thread; set `session.key: input.conversation_id` for history, the connector does not enable sessions), `conversation_type` (`personal`, `groupChat`, `channel`), `mentioned`, `sender` (`id`, `aad_object_id`, `name`), `team` and `channel` (`id`, `name`, or `null`), `message_id`, `raw` (full Teams activity). Files and images are not downloaded or passed as files; attachment details are only in `raw`.
- **Automatic outbound connector**: a run started by an inbound Teams connector on the same Connection gets a reply in that conversation (in channels, in the same thread). Otherwise it posts a new message in the default `channel`, which needs the app installed in that channel's team (connic learns where to post once the app is installed). Without either, the delivery fails. Plain text becomes the message; JSON may use `text`, `body`, `message`, `response`, or `output`. Teams renders Markdown.
- **Agent-tool / middleware outbound connector**: `text` is required; other fields are rejected. Same routing.
- Teams accepts at most 7 messages per second and 1,800 per hour per bot and conversation; when Teams asks to slow down, connic retries after the wait.

Full setup: [Microsoft Teams documentation](https://connic.co/docs/v1/connectors/microsoft-teams).

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
- **Reconnect**: append a client-chosen connection ID to the URL (`<websocket-url>/<connection id>`; `connected` returns it as `connection_id`); without one, connic generates an ID that can't be resumed. A connection with the ID of an open one replaces it: the older one gets `CONNECTION_REPLACED` and close code 4009 (connic can't tell a dropped connection from a live one, so the latest wins). Use one ID per tab or device and don't auto-reconnect after 4009. Runs keep going when a connection drops; answers that finished while the client was gone arrive right after `connected` on its next connection with the same ID, and answers that finish later arrive on the new connection. Only the final event is replayed (`response` or `stream_end` with `full_response`, or their `error` variants) with the original message `id`, no chunks or progress events. An answer counts as missed once the client closed the connection or reconnected; one sent on a connection that broke off unnoticed isn't resent. Replay only reaches the same client: any client with the secret when **Require Authentication** is on, the same end user of the same app on `apps_only` connectors (connection IDs are per end user there), and nobody when auth is off. Up to 20 missed answers per connection ID, kept for the session timeout after the latest one; an answer over 256 KiB arrives with the error `Response too large to replay`.
- **Conversation history** comes only from the agent's `session:` config, not from the connection. Set `session.key: input.conversation_id` (or a `context.*` key set in middleware) and send that field in every message; without a key, all conversations share one session. History outlives the connection: reconnecting with the same ID continues it.

## Pausing a connector

Pause a connector from its drawer in the Connector Flow (**Pause** / **Resume**) or with connic MCP `update_connector` and `enabled: false`. A paused connector starts no new runs; runs already queued or running are not affected. What happens to events during the pause depends on the type:

- **Kafka, SQS, RabbitMQ, Azure Service Bus**: connic stops reading. Messages stay in the topic, queue, or subscription and are processed after resume. Kafka continues from the consumer group's last committed offset (offsets are committed after the run is created), so messages must still be within topic retention.
- **MQTT**: connic disconnects. The broker holds QoS 1 messages in the session and sends them after resume, as long as the session hasn't expired; QoS 0 messages sent during the pause are lost.
- **Postgres**: connic stops listening. Notifications sent during the pause are lost.
- **Webhooks and messaging** (such as `webhook`, `stripe`, `slack`, `teams`, `github`, `gitlab`, `jira`, `linear`, `notion`, `pipedrive`, `shopify`, `telegram`, `twilio_messaging`, `whatsapp`, `google_pubsub` push): requests are acknowledged without starting runs. The events are not stored or replayed.
- **WebSocket**: new connections are refused; an open connection gets a `CONNECTOR_INACTIVE` error on its next message and closes.
- **Twilio Voice, Telnyx Voice**: incoming calls are rejected.
- **Cron**: no runs start; schedule times missed during the pause are not made up.

New inbound Kafka, SQS, RabbitMQ, MQTT, Azure Service Bus, and Postgres connectors created in the dashboard start paused; resume them once linked. Connectors declared in `connectors/*.yaml` start active. To stop intake during an incident or maintenance window without losing events, pause a Kafka, SQS, RabbitMQ, or Azure Service Bus connector.

## Linking a connector to multiple agents

Connectors that support multiple agents trigger every linked agent in parallel for each event. This supports fan-out, such as one webhook starting both an `intake` agent and an `audit` agent. Each agent runs independently with its own input, logs, and result. Twilio Voice and Telnyx Voice are single-agent connectors and do not support fan-out.

## Where the payload ends up

When a connector fires, the inbound event reaches `middleware/<agent>.py::before(content, context)` in two forms:

- **`content`** — the LLM-facing message: a dict with `role: "user"` and a list of `parts` (text and/or binary attachments). This is what the agent reasons over. Mutate it to attach documents, prepend context, redact PII.
- **`context["payload"]`** — the connector's normalized payload before user middleware transforms it: the JSON body of a webhook, GET query params, normalized multipart fields/files, parsed Kafka message plus metadata, email fields, etc. Read it for auth tokens, identity claims, routing metadata, and anything else you want middleware to see but the LLM should not.

Per-connector payload shapes are documented in the sections above. Normalize them in middleware when tools need one stable schema. See [the authentication walkthrough](tools-and-python.md#end-user-authentication-and-per-run-permissions) for using credentials from `context["payload"]`.
