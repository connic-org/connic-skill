# Connectors

Connectors define how agents are triggered, what input they receive, and where results go. Each connector can link to one or more agents; one trigger dispatches its input to every linked agent. There is **no** native Slack, Discord, GitHub, or Notion connector — use a `webhook` connector with your own forwarder, an MCP server, or a custom tool.

Use connectors to run agents from HTTP requests, queue messages, email, schedules, and calls from a backend. They provide provisioned endpoints, transport-specific authentication, sync/async modes, delivery rules, and fan-out. There is no generic inbound deduplication or replay guarantee; design idempotent consumers for transports that can redeliver. The REST API is for project management, not starting event-driven runs.

Connectors are configured per environment in the **Dashboard**, not in YAML. Each connector is linked to one or more agents. For inbound connectors, the incoming event becomes the agent's input. Outbound email and Telegram connectors consume the agent output; outbound webhook, Kafka, and SQS publish a full run envelope instead.

The full list and the modes each one supports:

| Connector | Modes | Use |
| --- | --- | --- |
| `cron` | Inbound | Schedule a recurring run with a fixed prompt. |
| `email` | Inbound (IMAP) / Outbound (SMTP) | Receive emails into the agent; send replies / new mail. |
| `kafka` | Inbound (Consumer) / Outbound (Producer) | Consume from / produce to a Kafka topic. |
| `mcp` | Inbound (Sync / Inbound) | Expose Connic agents as MCP tools to external MCP clients. |
| `postgres` | Inbound | LISTEN/NOTIFY-driven trigger. |
| `s3` | Inbound | React to S3 object events (via SNS/EventBridge). |
| `sqs` | Inbound (Consumer) / Outbound (Producer) | Consume from / produce to an SQS queue. |
| `stripe` | Inbound | React to Stripe webhook events. |
| `telegram` | Inbound / Outbound | Telegram bot — receive messages, send replies. |
| `webhook` | Inbound / Outbound / Sync | Generic HTTP. Sync = HTTP request/response; Inbound = fire-and-forget; Outbound = call out to your URL. |
| `websocket` | Sync (real-time chat) | Persistent bidirectional session. |

Common dashboard flow: open the agent's detail page → **+** on Connector Flow → **Create New Connector** → pick a type → configure → save. Supported connectors such as Postgres and outbound webhook can also reach private endpoints via **Connic Bridge** — set the Bridge in the connector config. Private MCP servers that an agent consumes use `mcp_servers[].bridge` in agent YAML instead.

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

Outbound: configure SMTP server / port / username / password / From address / From name, and optionally a Default Recipient. The agent's *output* is JSON with `to`, `subject`, `body`, and optional `cc`, `bcc`, `html_body`, `reply_to`. A recipient is required — it comes from the output's `to` or the connector's Default Recipient; with neither, the send fails. `subject` defaults to `"Agent Response"` when omitted. The connector does **not** automatically reply to the inbound sender — your agent must echo the right `to` (or rely on the Default Recipient). A bare (non-JSON) string is sent as the body, so it only delivers when a Default Recipient is configured.

## kafka

Inbound (Consumer) and Outbound (Producer) modes.

- **Connection**: environment-scoped dashboard fields for bootstrap servers, SASL credentials, topic name, and consumer group (inbound).
- **Inbound payload**: the parsed message value. Metadata is exposed at `_kafka` inside the payload: `{topic, partition, offset, timestamp, key}` — not in `context`. JSON-object values are dispatched with their top-level fields plus `_kafka`; anything else is wrapped under a `message` key — non-JSON values as `{"message": "<raw text>", "_kafka": ...}`, null values (compaction tombstones) as `{"message": null, "_kafka": ...}`. Tombstones DO trigger runs; use `_kafka.key` to identify the deleted entity.
- **Outbound**: publishes a run envelope with `run_id`, `agent_name`, `status`, `output`, `error`, `started_at`, `ended_at`, and `token_usage`. If the run was triggered by inbound Kafka, its key is preserved for partition ordering; otherwise the Kafka key falls back to `run_id`.
- **Outbound delivery gating** (applies to every outbound connector, not just Kafka): only runs that end `completed` are delivered — failed and cancelled runs are skipped. A run ended via `StopProcessing` counts as completed and IS published (the stop message becomes `output`) unless raised with `publish_outbound=False` — see [StopProcessing](tools-and-python.md). Returning `None`/empty output does not suppress delivery.

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
- **Outbound**: sends the same full run envelope as Kafka (`run_id`, `agent_name`, `status`, `output`, `error`, timestamps, and `token_usage`) to the configured queue.

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

- **Outbound**: the agent's output can be JSON with `text` (also accepts `message` or `body`) and optionally `chat_id`, or a bare string (sent as the message text). `chat_id` is **not** automatically resolved from the triggering run — either echo it from `input.chat_id` in the agent's output, or set a default Chat ID on the outbound connector; with neither, the send fails. Messages are always sent with `parse_mode: HTML` — the agent cannot override it.

There are no `telegram.send_message` or `telegram.send_photo` predefined tools. Sending photos / files / richer messages isn't supported by the outbound connector itself — for that, write a custom tool that hits the Telegram Bot API directly.

## webhook

The most flexible HTTP connector. Three independent modes — pick the one that matches your traffic pattern.

| Mode | What it does |
| --- | --- |
| **Sync (Request-Response)** | Caller `POST`s; connector blocks until the agent finishes and returns the result. 5-minute hard timeout. |
| **Inbound (Fire & Forget)** | Caller `POST`s; connector returns immediately with `{status, dispatched_to, run_ids[]}`. |
| **Outbound** | The full completed-run envelope is `POST`ed to a URL you configure, with `X-Connic-Signature` and `X-Connic-Timestamp` headers. |

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

For sync, the agent's response (string or — if `output_schema` is set — structured object) is what populates `result.output`. For inbound, the response is the dispatch confirmation; the run's output is visible in the dashboard. Outbound sends the same run envelope as Kafka and SQS (`run_id`, `agent_name`, `status`, `output`, `error`, timestamps, and `token_usage`). Verify its hex HMAC-SHA256 over `timestamp + "." + raw_body` using the connector signing secret and a constant-time comparison; reject timestamps outside a five-minute window.

## websocket

A single "Sync (Real-time Chat)" mode. The connector hosts a WS endpoint; each connection is a session through which messages flow.

- **Auth**: governed by the same **Require Authentication** toggle as the webhook connector (default on). When on, send `{"secret": "<connector secret>"}` as the first message after connecting, or pass `X-Connic-Secret` as a query param / header during the handshake. When off, the WS endpoint is open and authentication is your responsibility — typically a JWT in the first message that you verify in `middleware/<agent>.py::before`. Turn it off when each connection already carries a stronger per-user credential than a shared secret would provide.
- **Message protocol**: client sends the canonical `{type: "message", id?, payload: {message, context}}` envelope; shorthand `{"message": "..."}` and `{"content": "..."}` forms are also accepted. Server replies `ack` → `stream_start` → `stream_chunk` (multiple) → `stream_end` (with `full_response`, `token_usage`) when streaming is on; or a single `response` message when streaming is off. Agents with output guardrails still use the streaming event contract, but send one `stream_chunk` after the run completes so guardrails can inspect the full response before any text is released.
- **Files / multimodal**: the payload may carry a top-level `files` array in the same shape as the [webhook multipart normalization](#webhook) (`{name, mime_type, data: "<base64>", size}`); each entry becomes a binary part of the LLM-facing `content`. This path has no Connic MIME allowlist or per-file size cap, so provider limits apply. Validate uploads in `before` when needed.
- **Config**: streaming toggle, session timeout (60–86,400 seconds, default 3,600), max messages per session (1–10,000, default 100).
- **`connector_run_id`** is returned on connect and identifies the session.
- Conversation history persists only for that connection; closing it ends the session.

## Linking one connector to multiple agents

All agents linked to a single connector are triggered in parallel for each event. Useful for fan-out (a webhook hitting both an `intake` agent and an `audit` agent) — each linked agent runs independently with its own input, logs, and result.

## Where the payload ends up

When a connector fires, the inbound event reaches `middleware/<agent>.py::before(content, context)` in two forms:

- **`content`** — the LLM-facing message: a dict with `role: "user"` and a list of `parts` (text and/or binary attachments). This is what the agent reasons over. Mutate it to attach documents, prepend context, redact PII.
- **`context["payload"]`** — the connector's normalized payload before user middleware transforms it: the JSON body of a webhook, GET query params, normalized multipart fields/files, parsed Kafka message plus metadata, email fields, etc. Read it for auth tokens, identity claims, routing metadata, and anything else you want middleware to see but the LLM should not.

Per-connector payload shapes are documented in the sections above. Normalize them in middleware when tools need one stable schema. See [the authentication walkthrough](tools-and-python.md#end-user-authentication-and-per-run-permissions) for using credentials from `context["payload"]`.
