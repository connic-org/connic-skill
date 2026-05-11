# Connectors

Connectors are how external systems **trigger** an agent and how an agent reaches out to the world. They are the canonical (and, in practice, the only) way to fire a Connic agent from anything outside the platform — your backend, a queue, a scheduled job, a third-party webhook, an email. There are exactly **eleven** built-in connectors. There is **no** Slack, Discord, GitHub, Notion, etc. connector — use a `webhook` connector with your own forwarder, an MCP server, or a custom tool.

The REST API's `/v1/projects/<id>/agents/<name>/trigger` endpoint exists for internal admin and testing scenarios; it is **not** the production path for external integrations and should not be proposed as one. When someone asks "how does my backend call an agent?", the answer is "create a webhook connector and POST to its URL," not "use the REST API."

Connectors are configured in the **Dashboard**, not in YAML. Each connector is linked to one or more agents. For inbound connectors, the incoming event becomes the agent's `input` (the `content` passed to `middleware/<agent>.py::before`). For outbound modes, the agent's *output* (usually structured JSON) is consumed by the connector.

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

Common dashboard flow: open the agent's detail page → **+** on Connector Flow → **Create New Connector** → pick a type → configure → save. Some connectors (postgres, MCP server in inbound mode, etc.) can also reach private endpoints via **Connic Bridge** — set the Bridge in the connector config.

Auth on `webhook`, `websocket`, and `mcp` (server-mode) connectors is governed by a **Require Authentication** toggle (default on). When on, callers must present the connector secret as:

- `X-Connic-Secret: <secret>` header (preferred)
- `Authorization: Bearer <secret>` header
- `?secret=<secret>` query parameter (last resort — leaks in logs)

When off, the connector endpoint is open at the edge and authentication moves to your code — typically a JWT verified in `middleware/<agent>.py::before` (see the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions)). Use this when the inbound payload already carries a stronger per-user credential than a shared static secret would provide; leave the toggle on for the simple service-to-service case.

### Connectors don't have to fire an LLM

Any inbound connector can be linked to a [tool-type agent](agent-yaml.md#tool-agent) instead of an LLM-type agent. The connector payload is passed straight to your Python function as kwargs — no model in the loop. You still get run logs, retries, judges, and the rest of the platform machinery; you just skip the reasoning step. Use this for non-LLM consumers (Kafka → ingestion, S3 → ETL, webhook → routing) where the work is deterministic and you only want the connector and observability layer.

## cron

Schedule a recurring run.

- **Schedule**: standard cron syntax. **All schedules are evaluated in UTC** — convert local time before configuring. There's no per-connector timezone.
- **Prompt**: the dashboard takes a single text **prompt** (not arbitrary JSON). That prompt is what the agent receives as input.
- **Inbound payload shape**: `{"trigger": "cron", "schedule": "<cron expr>", "triggered_at": "<iso>", "prompt": "<your prompt>"}`.

## email

IMAP inbound + SMTP outbound. **You bring your own mailbox credentials** — Connic does not provision an email address. Inbound and outbound are configured as **separate connectors** (different mode), both linked to the same agent.

Inbound config: IMAP server / port / username / password (per env-vars), plus optional filters (unread-only, by sender/subject, mark-as-read).

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
  "_email": { /* connector metadata */ }
}
```

Attachments over 10 MB are listed as metadata only (no content). Field names are `filename`, `content_type`, `content` — not `name`, `mime_type`, `data`.

Outbound: configure SMTP server / port / username / password / From address / From name. The agent's *output* must be JSON with required `to`, `subject`, `body`, and optional `cc`, `bcc`, `html_body`, `reply_to`. The connector does **not** automatically reply to the inbound sender — your agent must echo the right `to`. Returning a bare string from the agent does not work.

## kafka

Inbound (Consumer) and Outbound (Producer) modes.

- **Connection** (per env-vars): bootstrap servers, SASL credentials, topic name, consumer group (inbound).
- **Inbound payload**: the parsed message value. Metadata is exposed at `_kafka` inside the payload: `{topic, partition, offset, timestamp, key}` — not in `context`.
- **Outbound**: agent emits a JSON object; the connector publishes it to the configured topic. If the run was triggered by an inbound Kafka message, the inbound message's key is preserved on the outbound for partition ordering.

## mcp (server mode)

Exposes Connic agents *as* MCP tools to external clients (Claude Desktop, IDEs, other MCP-aware tools).

- The connector provisions an MCP endpoint URL.
- Each agent linked to the connector becomes one MCP tool. The tool name is the agent name lowercased with underscores; the tool description is `"Invoke the <Agent Name> agent"`. The tool input schema is fixed: `{message: string (required), payload: object (optional)}`.
- Modes: **Sync** (recommended; returns the agent's result as the MCP tool result, 5-minute timeout) or **Inbound** (returns a run ID immediately).
- Supported JSON-RPC methods: `initialize`, `tools/list`, `tools/call`, `ping`.

**Don't confuse this with the `mcp_servers:` block in agent YAML** — that's the opposite direction (Connic agent as MCP *client*, calling external MCP tools).

## postgres

**Inbound only**, driven by Postgres `LISTEN/NOTIFY`. The connector subscribes to a channel; each `NOTIFY` becomes one agent run.

- **Config**: host, port, database, user, password, **channel** (the LISTEN channel name), SSL mode, an optional "Parse JSON Payload" flag. Reach private databases via Connic Bridge.
- **Inbound payload**: whatever was sent in the `NOTIFY` payload (parsed as JSON if the flag is on), plus a `_postgres` metadata block: `{channel, pid, timestamp}`.
- **Payload limit**: Postgres's NOTIFY ceiling is 8000 bytes — design publishers to send a key (e.g. record ID) and have the agent fetch the body via a custom tool.

There are no `postgres.query` / `postgres.fetch_one` tools. To read or write Postgres from inside an agent, write a custom tool with your DB driver of choice (asyncpg, psycopg).

## s3

**Inbound only**, driven by S3 object events. Wire S3 → SNS HTTP subscription → connector URL, **or** S3 → EventBridge → connector URL.

- **Config**: AWS access key + secret (no IAM role assumption documented), region, bucket, optional **prefix/suffix filters**, optional "Include Content" (downloads the object — text decoded as UTF-8, binary as base64), max file size 1–100 MB.
- **Inbound payload**: `{bucket, key, size, etag, event_name, event_time, content?, _s3: {event_source, aws_region, request_id, source_ip}}`.

There are no `s3.get_object` / `s3.put_object` / `s3.list_objects` tools. To upload/list/read from inside an agent, write custom tools using `boto3` / `aioboto3`.

## sqs

Inbound (Consumer) and Outbound (Producer).

- **Inbound config**: queue URL, AWS credentials, **visibility timeout** (default 300 s), **max messages** (1–10, default 10), **wait time** (long-polling, 0–20 s, default 20).
- **Inbound payload**: the message body (JSON-parsed when possible), plus `_sqs: {message_id, receipt_handle, queue_url, approximate_receive_count, sent_timestamp}`.
- **Delivery semantics**: successful runs delete the message; failed runs leave it for retry. FIFO queues are supported via a Message Group ID.
- **Outbound**: agent emits a JSON object; the connector sends it to the configured queue.

## stripe

**Inbound only**. The connector receives Stripe webhook events and verifies the signature.

- **Connic side**: configure a connector with a name and a **signing secret** (the `whsec_…` value from Stripe). Without the secret, events are rejected.
- **Stripe side**: in the Stripe Dashboard, create a webhook pointing at the connector's URL and pick the event types you want. **Event filtering happens in Stripe, not in Connic.** The Connic connector accepts whatever Stripe sends.
- **Inbound payload**: the parsed Stripe `Event` object.

## telegram

Telegram bot. **Inbound** and **Outbound** are separate connectors (different modes), both backed by the same bot token.

- **Inbound**: messages to the bot trigger runs.

  Payload shape:

  ```json
  {
    "update_id": ...,
    "text": "...",
    "chat_id": ...,
    "message": {
      "message_id": ..., "text": "...", "date": "...",
      "chat_id": ..., "chat_type": "private|group|...",
      "from_id": ..., "from_username": "...",
      "from_first_name": "...", "from_last_name": "..."
    },
    "raw": { /* original Telegram update */ }
  }
  ```

  There is no top-level `user_id` — the sender id is `message.from_id`. Optional `Allowed User IDs` allowlist gates which users the bot responds to. Inbound auth is verified by Connic via the `X-Telegram-Bot-Api-Secret-Token` header that Telegram sends.

- **Outbound**: the agent's output must be **JSON** with required `text` (and optionally `chat_id`, `parse_mode`). `chat_id` is **not** automatically resolved from the triggering run — either echo it from `input.chat_id` in the agent's output, or set a default Chat ID on the outbound connector. Returning a bare string from the agent does not send a message.

There are no `telegram.send_message` or `telegram.send_photo` predefined tools. Sending photos / files / richer messages isn't supported by the outbound connector itself — for that, write a custom tool that hits the Telegram Bot API directly.

## webhook

The most flexible HTTP connector. Three independent modes — pick the one that matches your traffic pattern.

| Mode | What it does |
| --- | --- |
| **Sync (Request-Response)** | Caller `POST`s; connector blocks until the agent finishes and returns the result. 5-minute hard timeout. |
| **Inbound (Fire & Forget)** | Caller `POST`s; connector returns immediately with `{status, dispatched_to, run_ids[]}`. |
| **Outbound** | The agent's *output* is `POST`ed to a URL you configure (with `X-Connic-Signature` HMAC-SHA256 + `X-Connic-Timestamp` headers). |

Inbound and Sync also accept `GET` (query params become the payload) and `multipart/form-data` (file uploads up to 10 MB; images, PDFs, Office docs etc. are passed as inline data to the LLM).

For multipart, `context["payload"]` is normalised to the same shape `trigger_agent` uses for [passing files](predefined-tools.md#passing-files-to-the-triggered-agent):

```python
{
  "fields":  {"customer_id": "cus_123", "tag": ["a", "b"]},   # text parts; repeated keys become lists
  "files":   [{"name": "invoice.pdf", "field_name": "file",
               "mime_type": "application/pdf",
               "data": "<base64>", "size": 12345}],
  "message": "..."  # copied from fields["message"] or fields["text"] if either was sent
}
```

If no supported file made it through, the payload collapses to just the `fields` dict (no `files`, no `message` key). File parts are filtered to a fixed allowlist (images, PDF, text/CSV/JSON/XML, Office, ODF, EPUB) and capped at 10 MB per file — anything else is silently dropped with a server-side log warning, not surfaced as a 4xx. Validate in `before` if you need to reject the request when an expected upload is missing.

The LLM-facing `content` is reconstructed automatically: each `files[*]` entry becomes a binary part, with `message` (or `text`, or a synthesised "Please analyze the attached file(s).") as the leading text part.

**Authentication** (Inbound / Sync) is controlled by a **Require Authentication** toggle on the connector. When on (the default), callers must present the connector secret as `X-Connic-Secret: <secret>`, `Authorization: Bearer <secret>`, or `?secret=<secret>` — requests without it are rejected at the edge. When off, the endpoint is open and authentication is your responsibility downstream: typically you verify a JWT or another credential inside `middleware/<agent>.py::before` and reject from there (see the [end-user authentication pattern](tools-and-python.md#end-user-authentication-and-per-run-permissions)). Turn it off when the inbound payload already carries a stronger credential than a static shared secret — JWTs, OIDC tokens, signed webhooks from a known upstream — so you're not stacking two redundant secrets. Leave it on for the simple case where the caller is just another service of yours.

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

For sync, the agent's response (string or — if `output_schema` is set — structured object) is what populates `result.output`. For inbound, the response is the dispatch confirmation; the run's output is visible in the dashboard.

## websocket

A single "Sync (Real-time Chat)" mode. The connector hosts a WS endpoint; each connection is a session through which messages flow.

- **Auth**: governed by the same **Require Authentication** toggle as the webhook connector (default on). When on, send `{"secret": "<connector secret>"}` as the first message after connecting, or pass `X-Connic-Secret` as a query param / header during the handshake. When off, the WS endpoint is open and authentication is your responsibility — typically a JWT in the first message that you verify in `middleware/<agent>.py::before`. Turn it off when each connection already carries a stronger per-user credential than a shared secret would provide.
- **Message protocol**: client sends `{type: "message", id, payload: {message, context}}`. Server replies `ack` → `stream_start` → `stream_chunk` (multiple) → `stream_end` (with `full_response`, `token_usage`) when streaming is on; or a single `response` message when streaming is off.
- **Config**: streaming toggle, session timeout (default 1 h), max messages per session (default 100).
- **`connector_run_id`** is returned on connect and identifies the session.

## Linking one connector to multiple agents

All agents linked to a single connector are triggered in parallel for each event. Useful for fan-out (a webhook hitting both an `intake` agent and an `audit` agent) — each linked agent runs independently with its own input, logs, and result.

## Where the payload ends up

When a connector fires, the inbound event reaches `middleware/<agent>.py::before(content, context)` in two forms:

- **`content`** — the LLM-facing message: a dict with `role: "user"` and a list of `parts` (text and/or binary attachments). This is what the agent reasons over. Mutate it to attach documents, prepend context, redact PII.
- **`context["payload"]`** — the **raw, untransformed connector payload**: the JSON body of a webhook, GET query params, the parsed Kafka message, the email metadata, etc. Read it for auth tokens, identity claims, routing metadata, and anything else you want middleware to see but the LLM should not.

Per-connector payload shapes are documented in the sections above. If you need a stable schema across multiple connectors, normalize in middleware before tools see it. The auth-token-in-payload pattern is the most common reason to reach for `context["payload"]` — see [the end-user authentication walkthrough](tools-and-python.md#end-user-authentication-and-per-run-permissions).
