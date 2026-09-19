# Connectors

Connectors define how agents are triggered, what input they receive, and where results go. Most connectors can link to one or more agents; one trigger dispatches its input to every linked agent. SIP Voice and Twilio Voice each link to exactly one voice agent. Slack is native. Discord, GitHub, and Notion are not connector types — use a `webhook` connector with your own forwarder, an MCP server, or a custom tool.

Use connectors to run agents from HTTP requests, queue messages, email, schedules, and calls from a backend. They provide provisioned endpoints, transport-specific authentication, sync/async modes, delivery rules, and fan-out. There is no generic inbound deduplication or replay guarantee; design idempotent consumers for transports that can redeliver. The REST API is for project management, not starting event-driven runs.

Connectors are configured per environment in the **Dashboard**, not in YAML. For inbound connectors, the incoming event becomes the agent's input. Automatic outbound connectors for Email, Telegram, Slack, and Twilio Messaging can consume the final agent output; automatic outbound connectors for webhook, Kafka, and SQS publish a full run envelope instead. Agent-tool and middleware outbound connectors use a connector-owned payload schema and do not constrain the final response.

The full list and the modes each one supports:

| Connector | Modes | Use |
| --- | --- | --- |
| `cron` | Inbound | Schedule a recurring run with a fixed prompt. |
| `email` | Inbound (IMAP) / Outbound (SMTP) | Receive emails into the agent; send replies / new mail. |
| `kafka` | Inbound (Consumer) / Outbound (Producer) | Consume from / produce to a Kafka topic. |
| `mcp` | Inbound (Sync / Inbound) | Expose Connic agents as MCP tools to external MCP clients. |
| `postgres` | Inbound | LISTEN/NOTIFY-driven trigger. |
| `s3` | Inbound | React to S3 object events (via SNS/EventBridge). |
| `sip` | Sync (incoming voice) | Receive calls from a phone provider or phone system over SIP. |
| `sqs` | Inbound (Consumer) / Outbound (Producer) | Consume from / produce to an SQS queue. |
| `slack` | Inbound (Mentions) / Outbound | Trigger from bot mentions; reply to a thread or post to a channel. |
| `stripe` | Inbound | React to Stripe webhook events. |
| `telegram` | Inbound / Outbound | Telegram bot — receive messages, send replies. |
| `twilio_messaging` | Inbound / Outbound | Receive and send SMS/MMS, WhatsApp, and RCS messages. |
| `twilio` | Sync (incoming voice) | Connect an existing Twilio phone number to one deployed voice agent. |
| `webhook` | Inbound / Outbound / Sync | Generic HTTP. Sync = HTTP request/response; Inbound = fire-and-forget; Outbound = call out to your URL. |
| `websocket` | Sync (real-time chat) | Persistent bidirectional session. |

Common dashboard flow: open the agent's detail page → **+** on Connector Flow → **Create New Connector** → pick a type → configure → save. Supported connectors such as Postgres and outbound webhook can also reach private endpoints via **Connic Bridge** — set the Bridge in the connector config. Private MCP servers that an agent consumes use `mcp_servers[].bridge` in agent YAML instead.

## Outbound connector modes

An outbound connector has one mode:

- **Automatic outbound connector** — sends once after a completed run. Choose **All runs**, or choose **Only selected inputs** to send only when `run.connector_id` matches one of the selected inbound or sync connector IDs. All runs also includes manual, cron, and `trigger_agent` runs with no inbound source. Existing links without outbound connector settings remain **Automatic / All runs**.
- **Agent-tool outbound connector** — injects an editable connector tool into an LLM agent. Its default `action_name` is `send_to_<normalized connector name>`. Keep it unique across that agent's custom, predefined, connector, and MCP tools. The model decides whether and when to call it and may call several agent-tool outbound connectors in one run.
- **Middleware outbound connector** — project code calls it with `await send_connector(action_name, payload)`, and the model cannot access it.

Source filters and `StopProcessing(..., publish_outbound=False)` apply only to automatic outbound connectors. The flag does not undo an agent-tool or middleware outbound connector call.

Deployment tests do not deliver outbound messages. Automatic outbound connectors are suppressed, while calls to agent-tool and middleware outbound connectors are recorded as mocked tool calls for assertions.

Each outbound connector owns its payload schema, credentials, destination resolution, wire formatting, retries, and Bridge routing. The model and middleware receive no stored secrets or destination values. Agent-tool and middleware outbound connector payloads are:

| Connector | Payload |
| --- | --- |
| HTTP Webhook | `{"payload": { ... }}` |
| Kafka | `{"payload": { ... }, "key": "optional-string"}` |
| SQS | `{"payload": { ... }}` |
| Email | `{"body": "required", "to"?: string \| string[], "subject"?: string, "html_body"?: string, "cc"?: string \| string[], "bcc"?: string \| string[], "reply_to"?: string}` |
| Telegram | `{"text": "required", "chat_id"?: string \| integer}` |
| Slack | `{"text": "required", "channel_id"?: string, "thread_ts"?: string}` |
| Twilio Messaging | `{"text"?: string, "to"?: string, "media_urls"?: string[], "content_sid"?: string, "content_variables"?: object}`; supply text, media, or template content |

Unknown top-level payload fields are rejected. Routing fields are optional when the connector has a configured default or trusted matching inbound origin. For automatic outbound connectors, the final run output keeps the legacy connector-specific contract documented below. `output_schema` constrains only that final response; it does not combine or replace outbound connector payload schemas.

Custom domains apply to HTTP Webhook, MCP Server, S3, Stripe, Telegram, and WebSocket connector URLs.

Auth on `webhook`, `websocket`, and `mcp` (server-mode) connectors is governed by a **Require Authentication** toggle (default on). Accepted secret forms differ by transport:

- **Webhook:** `X-Connic-Secret` header (preferred), `Authorization: Bearer`, or `?secret=` query parameter.
- **WebSocket:** either header, `?secret=` / `?X-Connic-Secret=` during the handshake, or `{"secret": "..."}` as the first message.
- **MCP:** `Authorization: Bearer` or `X-Connic-Secret` header; query-string secrets are not accepted.

When off, the connector accepts requests without its shared secret. Verify the caller in `middleware/<agent>.py::before`, for example with a JWT or signed payload (see the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions)).

### Connectors don't have to fire an LLM

Any inbound connector can be linked to a [tool-type agent](agent-yaml.md#tool-agent) instead of an LLM-type agent. The normalized connector payload is passed as one dict to the function's required `payload` parameter, plus `context` when declared — payload keys are never expanded into keyword arguments. The run retains logs, configured retries, and judges without a reasoning step. Use this for deterministic consumers such as Kafka ingestion, S3 transforms, or webhook routing.

## cron

Schedule a recurring run.

- **Schedule**: standard cron syntax. **All schedules are evaluated in UTC** — convert local time before configuring. There's no per-connector timezone.
- **Prompt**: optional single text prompt (not arbitrary JSON). When configured, it is included in every scheduled payload.
- **Inbound payload shape**: `{"trigger": "cron", "schedule": "<cron expr>", "triggered_at": "<iso>"}`; `prompt` is added only when configured.

## email

IMAP inbound + SMTP outbound. **You bring your own mailbox credentials** — Connic does not provision an email address. Inbound and outbound are configured as **separate connectors** (different mode), both linked to the same agent.

Inbound config: IMAP server / port / username / password, mailbox (default `INBOX`), plus optional filters (unread-only, by sender/subject, mark-as-read).

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

Attachments over 10 MB are listed as metadata only (no content). Supported content includes common images, PDF/text/data formats, and DOCX/XLSX/PPTX; tracking pixels, tiny inline/signature images, and unknown formats are filtered out. Field names are `filename`, `content_type`, `content` — not `name`, `mime_type`, `data`.

Automatic outbound connector behavior: configure SMTP server / port / username / password / From address / From name, and optionally a Default Recipient. The agent's final *output* is JSON with `to`, `subject`, `body`, and optional `cc`, `bcc`, `html_body`, `reply_to`. A bare string becomes the body. Agent-tool and middleware outbound connectors instead use the Email payload schema above; `body` is required. Recipient precedence is explicit `to`, Default Recipient, then trusted matching inbound email context. Without an explicit subject, replies reuse the inbound subject with `Re:`; other sends use `"Agent Response"`.

## kafka

Inbound (Consumer) and Outbound (Producer) modes.

- **Connection**: environment-scoped dashboard fields for bootstrap servers, SASL credentials, topic name, and consumer group (inbound).
- **Inbound payload**: the parsed message value. Metadata is exposed at `_kafka` inside the payload: `{topic, partition, offset, timestamp, key}` — not in `context`. JSON-object values are dispatched with their top-level fields plus `_kafka`; anything else is wrapped under a `message` key — non-JSON values as `{"message": "<raw text>", "_kafka": ...}`, null values (compaction tombstones) as `{"message": null, "_kafka": ...}`. Tombstones DO trigger runs; use `_kafka.key` to identify the deleted entity.
- **Automatic outbound connector**: publishes a run envelope with `run_id`, `agent_name`, `status`, `output`, `error`, `started_at`, `ended_at`, and `token_usage`. If the run was triggered by inbound Kafka, its key is preserved for partition ordering; otherwise the Kafka key falls back to `run_id`. An agent-tool or middleware outbound connector publishes its `payload` as the message value and may set `key` explicitly.
- **Automatic outbound connector gating** (applies to every outbound connector, not just Kafka): only runs that end `completed` are delivered — failed and cancelled runs are skipped. A run ended via `StopProcessing` counts as completed and is published (the stop message becomes `output`) unless raised with `publish_outbound=False` — see [StopProcessing](tools-and-python.md). Returning `None` or empty output does not suppress the automatic outbound connector.

## mcp (server mode)

Exposes Connic agents *as* MCP tools to external clients (Claude Desktop, IDEs, other MCP-aware tools).

- The connector provisions an MCP endpoint URL.
- Each agent linked to the connector becomes one MCP tool. The tool name is lowercased, with spaces and hyphens replaced by underscores; the tool description is `"Invoke the <Agent Name> agent"`. The tool input schema is fixed: `{message: string (required), payload: object (optional)}`. Keys from `payload` are merged into the agent input alongside `message`.
- Modes: **Sync** (recommended; returns the agent's result as the MCP tool result, 5-minute timeout) or **Inbound** (returns a run ID immediately; this is Connic fire-and-forget behavior, not MCP Tasks).
- Protocols: stateless `2026-07-28`; Streamable HTTP `2025-11-25`, `2025-06-18`, and `2025-03-26`; and HTTP/SSE `2024-11-05`.
- Authentication uses the connector's pre-shared secret in `Authorization: Bearer` or `X-Connic-Secret`; it has no MCP OAuth discovery or interactive authorization flow. Requests with a browser `Origin` header are rejected, so use a native or server-side MCP client.

**Don't confuse this with the `mcp_servers:` block in agent YAML** — that's the opposite direction (Connic agent as MCP *client*, calling external MCP tools). See [MCP servers](guardrails-schemas-mcp.md#mcp-servers).

## postgres

**Inbound only**, driven by Postgres `LISTEN/NOTIFY`. The connector subscribes to a channel; each `NOTIFY` becomes one agent run.

- **Config**: host, port, database, user, password, **channel** (the LISTEN channel name), SSL mode, an optional "Parse JSON Payload" flag. Reach private databases via Connic Bridge.
- **Inbound payload**: includes `_postgres: {channel, pid, timestamp}`. With JSON parsing enabled, an object keeps its top-level fields; a list or scalar is wrapped as `{data: <value>, _postgres: ...}`. With parsing off, invalid JSON, or plain text, it is `{message: <text>, _postgres: ...}`.
- **Payload limit**: Postgres's NOTIFY ceiling is 8000 bytes — design publishers to send a key (e.g. record ID) and have the agent fetch the body via a custom tool.

There are no `postgres.query` / `postgres.fetch_one` tools. To read or write Postgres from inside an agent, write a custom tool with your DB driver of choice (asyncpg, psycopg).

## s3

**Inbound only**, driven by S3 object events. Wire S3 → SNS HTTP subscription → connector URL, **or** S3 → EventBridge → connector URL.

- **Config**: AWS access key and secret, region, bucket, event mode (**Object Created** by default, or **All Events** to include deletes/restores), optional **prefix/suffix filters**, optional "Include Content", and max file size 1–100 MB.
- **Inbound payload**: `{bucket, key, size, etag, event_name, event_time, content?, _s3: {event_source, aws_region, request_id, source_ip}}`. When content is included it is `{text, content_type, size_bytes, encoding}`; UTF-8 text uses `encoding: "utf-8"` and binary uses base64.
- **SNS / EventBridge setup**: send to `<connector-url>?secret=<secret-key>`. SNS subscriptions can confirm against this URL; EventBridge can use it as an API Destination.

There are no `s3.get_object` / `s3.put_object` / `s3.list_objects` tools. To upload/list/read from inside an agent, write custom tools using `boto3` / `aioboto3`.

## sqs

Inbound (Consumer) and Outbound (Producer).

- **Inbound config**: queue URL, AWS credentials, **visibility timeout** (6–43,200 s, default 300), **max messages** (1–10, default 10), **wait time** (long-polling, 0–20 s, default 20).
- **Inbound payload**: a JSON object keeps its top-level fields; any non-object body is wrapped under `message`. `_sqs: {message_id, receipt_handle, queue_url, approximate_receive_count, sent_timestamp}` is added in both cases.
- **IAM**: inbound consumers need `sqs:ReceiveMessage`, `sqs:DeleteMessage`, and `sqs:ChangeMessageVisibility`; outbound producers need `sqs:SendMessage`.
- **Delivery semantics**: the message is deleted only after **all linked agent runs** succeed; if any fails, it remains for retry. Connic tracks dispatched runs and extends message visibility while they are still running. FIFO queues are supported via a Message Group ID.
- **Automatic outbound connector**: sends the same full run envelope as Kafka (`run_id`, `agent_name`, `status`, `output`, `error`, timestamps, and `token_usage`) to the configured queue. An agent-tool or middleware outbound connector sends its nested `payload` as the message body.

## stripe

**Inbound only**. The connector receives Stripe webhook events and verifies the signature.

- **Connic side**: configure a connector with a name and a **signing secret** (the `whsec_…` value from Stripe). Without the secret, events are rejected.
- **Stripe side**: in the Stripe Dashboard, create a webhook pointing at the connector's URL and pick the event types you want. **Event filtering happens in Stripe, not in Connic.** The Connic connector accepts whatever Stripe sends.
- **Inbound payload**: the parsed Stripe `Event` object.

## telegram

Telegram bot. **Inbound** and **Outbound** are separate connectors (different modes), both backed by the same bot token.

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

  Photos, voice messages, audio, videos, video notes, documents, and animations are downloaded when available. The largest photo size is used. Each downloaded item appears in top-level `files` as `{name, mime_type, data, size}`, with `data` base64-encoded.

  There is no top-level `user_id` — the sender id is `message.from_id`. Optional `Allowed User IDs` allowlist gates which users the bot responds to. Inbound auth is verified by Connic via the `X-Telegram-Bot-Api-Secret-Token` header that Telegram sends.

- **Automatic outbound connector**: the final output can be JSON with `text` (also accepts `message` or `body`) and optionally `chat_id`, or a bare string. Agent-tool and middleware outbound connectors use required `text` plus optional `chat_id`. An explicit chat wins, followed by the connector's default Chat ID, then trusted matching inbound Telegram context. Messages use `parse_mode: HTML`.

There is no universal `telegram.send_message` or `telegram.send_photo` predefined tool. An agent-tool outbound connector for Telegram injects only its chosen `action_name`. This outbound connector does not support photos, files, or richer messages; use a custom tool for those Telegram Bot API methods.

## sip voice

SIP Voice is in beta. It connects incoming calls from a phone provider or phone system to one deployed agent with `voice_config`. It runs in sync mode and does not place outbound calls. The provider must offer G.711 μ-law or A-law audio.

Choose one connection setup:

- **Connic logs in to the provider:** enter the provider's SIP server, username, password, and transport. Add a separate authentication username or outbound proxy only when the provider supplies one. Connic registers with the provider, while the required IP or CIDR allowlist identifies the servers permitted to send calls.
- **The provider sends calls to Connic:** route the number to the generated SIP address. Authenticate incoming calls with trusted provider IPs or the generated SIP username and password.

Each connector selects a called number or SIP username and links it to one deployed voice agent. A saved connection can serve several destinations, but every destination must be unique within that connection, including across environments. Disabling or deleting a connector stops new calls to its destination. Run details show agent speech, tools, timings, errors, and user speech when transcription is enabled.

Full setup: [SIP Voice documentation](https://connic.co/docs/v1/connectors/sip).

## twilio messaging

Use separate inbound and outbound `twilio_messaging` connectors for SMS/MMS, WhatsApp, or RCS. Both reuse saved `twilio` connections with Voice: Account SID, region, and regional Auth Token. US1 supports all three channels; IE1 supports SMS text only and excludes +1 senders and recipients; AU1 supports Voice only.

- **Config**: required `connection_id`, `mode`, `channel` (`sms`, `whatsapp`, or `rcs`), and `sender`. Sender formats are `+E164`, `whatsapp:+E164`, or `rcs:sender-id`; Connic adds omitted channel prefixes. Outbound settings optionally include `to`, `messaging_service_sid`, `content_sid`, and `content_variables` (a JSON object).
- **Inbound setup**: Connic registers the generated webhook automatically for SMS/MMS numbers and registered WhatsApp or RCS senders. It removes the old webhook when the sender changes or the connector is deleted, but only if the webhook still belongs to that connector. The WhatsApp Sandbox requires manual setup: copy the generated URL into **When a Message Comes in**, select HTTP POST, and update or remove it manually later. Messaging setup leaves Voice handling unchanged.
- **Inbound payload**: `text`, `from`, `to`, `channel`, `message_sid`, and a stable `conversation_id`. Downloaded media uses standard Connic `files` input, limited to 10 MiB total per message.
- **Conversation history**: explicitly configure the agent with `session: {key: input.conversation_id, ttl: 86400}` and redeploy it. The connector does not enable sessions automatically.
- **Outbound content**: automatic delivery accepts a plain string or the structured payload above. Agent-tool and middleware delivery uses the structured payload. Text has a 1,600-character limit; media URLs must be public HTTP/HTTPS URLs without embedded credentials and satisfy the selected channel's media limits.
- **Recipient precedence**: explicit payload `to`, configured default `to`, then the customer from trusted matching inbound context. Reply fallback requires the same saved connection, account, region, channel, and Twilio sender. RCS recipient addresses use `rcs:+E164`, without automatic SMS fallback.
- **WhatsApp templates**: outside the 24-hour customer service window, use an approved `content_sid` (`HX…`) with `content_variables`. Explicit template content cannot be combined with `text` or `media_urls`. A configured Content SID ignores plain generated text; send variables to fill the template.
- **Delivery status**: a completed outbound connector run means Twilio accepted the API request. Check Twilio Messaging Logs for final delivery status.

Full setup: [Twilio Messaging documentation](https://connic.co/docs/v1/connectors/twilio-messaging).

## twilio voice

Twilio Voice is in beta. It connects incoming calls on an existing customer-owned Twilio number to one deployed agent with `voice_config`. It runs in sync mode and does not place outbound calls.

1. Deploy a voice agent using a supported realtime model.
2. In Twilio, open a voice-capable number and confirm its incoming-call region. Clear any existing incoming-call webhook, TwiML Application, or SIP trunk in that region.
3. In Connic, create a Twilio connection with the Account SID, region, and that region's Auth Token. Supported regions are US1, IE1, and AU1; US1 is the default.
4. Add the Twilio Voice connector to the agent, reuse the saved connection, and select the number. Connic installs the number's incoming-call webhook with HTTP POST.

The Auth Token must match the selected region; an API key SID and secret are not accepted in its place. One connection can be reused for other numbers in the same account and region. Changing numbers installs the new webhook and removes Connic's webhook from the previous number. Deleting the connector removes the webhook only while it still points to Connic.

Run details show the conversation transcript when transcription is available, tool calls, timings, and errors. Voice configuration and limitations are in [agent-yaml.md](agent-yaml.md#voice-llm-agent).

## slack

Inbound Slack connectors trigger linked agents from signed `app_mention` events. Origin-thread replies require the inbound and outbound connectors to use the same verified Slack Connection.

Configure Slack in the Dashboard: create a Slack Connection, open the generated app manifest, save the Signing Secret, verify the Events request URL, install the app, and save its `xoxb-` Bot User OAuth Token. Then create the inbound or outbound connector and invite the bot to every channel where it should receive mentions or post messages. Connic MCP can read Slack connectors but cannot create them because creation requires this Connection setup flow.

One Slack Connection can have one inbound connector. Link additional agents to that connector; outbound connectors can reuse the Connection.

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

Inbound and Sync also accept `GET` (query params become the payload, with the authentication `secret` stripped), `application/x-www-form-urlencoded`, and `multipart/form-data` (file uploads up to 10 MB; images, PDFs, Office docs etc. are passed as inline data to the LLM).

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

If no supported file is accepted, the payload contains only the top-level form fields and no `files` key. File parts use a fixed allowlist (images, PDF, text/CSV/JSON/XML, Office, ODF, EPUB) and a 10 MB per-file limit. Unsupported or oversized parts are omitted. Validate in `before` if a missing upload should reject the request.

The LLM-facing `content` is reconstructed automatically: each `files[*]` entry becomes a binary part, and the leading text part is the payload with only `files` removed. A `{message, files}` payload renders as the plain `message` string; any richer shape is JSON-serialised.

**Authentication** (Inbound / Sync) is controlled by a **Require Authentication** toggle on the connector. When on (the default), callers must present the connector secret as `X-Connic-Secret: <secret>`, `Authorization: Bearer <secret>`, or `?secret=<secret>`; unauthenticated requests are rejected. When off, verify the caller in `middleware/<agent>.py::before`, for example with a JWT or signed payload (see the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions)).

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

A single "Sync (Real-time Chat)" mode. The connector hosts a WS endpoint; each connection is a session through which messages flow.

- **Auth**: governed by the same **Require Authentication** toggle as the webhook connector (default on). When on, send `{"secret": "<connector secret>"}` as the first message after connecting, or pass `X-Connic-Secret` as a query param / header during the handshake. When off, the WS endpoint is open and authentication is your responsibility — typically a JWT in the first message that you verify in `middleware/<agent>.py::before`. Turn it off when each connection already carries a stronger per-user credential than a shared secret would provide.
- **Message protocol**: client sends the canonical `{type: "message", id?, payload: {message, context}}` envelope; shorthand `{"message": "..."}` and `{"content": "..."}` forms are also accepted. Server replies `ack` → `stream_start` → `stream_chunk` (multiple) → `stream_end` (with `full_response`, `token_usage`) when streaming is on; or a single `response` message when streaming is off. Agents with output guardrails still use the streaming event contract, but send one `stream_chunk` after the run completes so guardrails can inspect the full response before any text is released.
- **Files / multimodal**: the payload may carry a top-level `files` array in the same shape as the [webhook multipart normalization](#webhook) (`{name, mime_type, data: "<base64>", size}`); each entry becomes a binary part of the LLM-facing `content`. This path has no Connic MIME allowlist or per-file size cap, so provider limits apply. Validate uploads in `before` when needed.
- **Config**: streaming toggle, session timeout (60–86,400 seconds, default 3,600), max messages per session (1–10,000, default 100).
- **`connector_run_id`** is returned on connect and identifies the session.
- Conversation history persists only for that connection; closing it ends the session.

## Linking a connector to multiple agents

Connectors that support multiple agents trigger every linked agent in parallel for each event. This supports fan-out, such as one webhook starting both an `intake` agent and an `audit` agent. Each agent runs independently with its own input, logs, and result. SIP Voice and Twilio Voice are single-agent and do not support fan-out.

## Where the payload ends up

When a connector fires, the inbound event reaches `middleware/<agent>.py::before(content, context)` in two forms:

- **`content`** — the LLM-facing message: a dict with `role: "user"` and a list of `parts` (text and/or binary attachments). This is what the agent reasons over. Mutate it to attach documents, prepend context, redact PII.
- **`context["payload"]`** — the connector's normalized payload before user middleware transforms it: the JSON body of a webhook, GET query params, normalized multipart fields/files, parsed Kafka message plus metadata, email fields, etc. Read it for auth tokens, identity claims, routing metadata, and anything else you want middleware to see but the LLM should not.

Per-connector payload shapes are documented in the sections above. Normalize them in middleware when tools need one stable schema. See [the authentication walkthrough](tools-and-python.md#end-user-authentication-and-per-run-permissions) for using credentials from `context["payload"]`.
