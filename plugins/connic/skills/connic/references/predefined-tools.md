# Predefined tools

Built-in tools that ship with the SDK. Reference them in an agent's `tools:` list by bare name — no module prefix.

```yaml
tools:
  - retrieval_query
  - trigger_agent
  - db_find
  - file_list
  - web_search
```

You can also call them from your own Python tools by importing from `connic.tools`:

```python
from connic.tools import retrieval_query, db_find, trigger_agent, web_search
```

## Agent orchestration

### `trigger_agent(agent_name, payload, wait_for_response=True, timeout_seconds=60)`

Run another agent in the same project and environment.

```python
result = await trigger_agent(
    "billing-assistant",
    payload={"invoice_id": "INV-123"},
    wait_for_response=True,
    timeout_seconds=60,
)
# {"run_id": "...", "status": "completed"|"failed"|"blocked"|"cancelled"|"awaiting_approval"|"timeout", "response": ..., "error": ...?}
```

Pass a dict or list for structured JSON. Strings can contain plain text or encoded JSON.

When `wait_for_response=False`, it returns immediately with just `run_id` (no `status` or `response`).

`timeout_seconds` has no upper limit of its own, but the calling run's `timeout` still applies. When the wait ends, the child run keeps running under its own timeout and the result has `status: "timeout"`.

Tests can assert on agents invoked through `trigger_agent` or `trigger_agent_at` with `expected_child_agents` in `tests/*.yaml`. See [cli-and-dev.md](cli-and-dev.md#asserting-on-triggered-agents).

#### Passing files to the triggered agent

`trigger_agent` has no `files` parameter. To send files, include a top-level `files` list in its payload:

```python
import base64
from connic.tools import trigger_agent

with open("/tmp/invoice.pdf", "rb") as f:
    encoded = base64.b64encode(f.read()).decode("ascii")

result = await trigger_agent(
    "invoice-extractor",
    payload={
        "message": "Extract totals from this invoice.",   # becomes the leading text part
        "files": [
            {
                "data": encoded,                          # MUST be base64
                "mime_type": "application/pdf",
                "name": "invoice.pdf",
            }
        ],
        "customer_id": "cus_123",                         # optional structured data
    },
    wait_for_response=True,
)
```

Shape rules:

- `files` is a list. Each entry needs `data` (base64-encoded bytes as an ASCII string), `mime_type`, and `name`.
- The receiving agent keeps every non-`files` key at the top level of `context["payload"]`.
- The receiving LLM sees the payload with only `files` removed. A `{message, files}` payload renders as the plain `message` string; any richer shape is JSON-serialised, so structured fields like `mode`, `data_points`, or `customer_id` are visible to the model.

Base64 increases payload size. For large files, pass a reference such as a presigned S3 URL, Retrieval `entry_id`, or project-database document ID and let the receiving agent fetch the binary through a tool.

### `trigger_agent_at(agent_name, payload, delay=None, unix_timestamp=None)`

Schedule a future run. **Maximum scheduling window is 30 days.** Use a cron connector + self-rescheduling pattern for longer windows.

```python
await trigger_agent_at(
    "followup",
    payload={"user_id": "u1"},
    delay={"h": 2, "m": 30},      # OR unix_timestamp=1730000000
)
# {"run_id": "...", "scheduled_at": "...", "status": "scheduled"}
```

`payload` accepts the same dict, list, and string forms as `trigger_agent`.

Provide **exactly one** of `delay` or `unix_timestamp`. A delay uses only `d`, `h`, `m`, and `s`, must be positive, and the absolute timestamp must be a future Unix timestamp. Scheduling is limited to 30 days. The tool returns immediately; `scheduled_at` is ISO 8601 in UTC.

## Retrieval

Managed semantic retrieval for each environment, optionally divided into namespaces. Use **dot-separated** names for hierarchy (e.g. `policies.hr.leave`, `products.pricing`), with a maximum depth of 10. Namespace scoping covers the whole subtree: querying `policies` also searches `policies.hr.leave` — the same inclusion rule deletes use.

```python
# Semantic search
result = await retrieval_query(
    query="refund policy",
    namespace="policies",        # optional; omit to search all namespaces
    min_score=0.3,               # default 0.3; clamped to 0.0–1.0
    max_results=3,               # default 3; clamped to 1–20
    metadata_filter={"source": "handbook"},  # optional; see "Metadata filters" below
)
# {"results": [{"score": 0.95, "content": "...", "metadata": {...}, "entry_id": "...", "namespace": "policies"}, ...]}

# Insert or update — indexing is async
await retrieval_store(
    content="Refunds are issued within 7 days.",
    entry_id="refund-policy",    # optional; auto-generated if omitted
    namespace="policies",
    metadata={"source": "handbook"},
)
# {"entry_id": "...", "job_id": "...", "status": "pending", "queued": true, "success": true}

# Delete — three modes (see below)
await retrieval_delete(entry_id="refund-policy", namespace="policies")            # single entry
await retrieval_delete(namespace="confluence",                                    # bulk by metadata
                       metadata_filter={"root_page_id": page_id,
                                        "run_id": {"$ne": current_run_id}})
await retrieval_delete(namespace="meetings.q1")                                   # wipe a subtree

# List namespaces — default depth=1; depth=0 returns all descendants (max depth 10)
top = await retrieval_list_namespaces(parent=None, depth=1)
# {"namespaces": [{"name": "policies", "entry_count": N, "total_entry_count": N, "has_children": true}, ...]}
children = await retrieval_list_namespaces(parent="policies", depth=1)
# {"parent": {...}, "namespaces": [...]}
```

`retrieval_query` inherits the current run's remaining deadline. If the deadline has already been reached or expires before a response arrives, it returns `{"error": "...", "results": []}` instead of matches.

Retrieval queries and metadata-filter deletes operate on committed index entries. A pending `retrieval_store` job is not searchable; for an existing namespace and `entry_id`, its committed content and metadata remain queryable until the replacement job commits atomically. Agents cannot poll a `job_id`, so schedule dependent cleanup as a later run with `trigger_agent_at` rather than inline.

### Metadata filters on `retrieval_query` and `retrieval_delete`

Both tools accept an optional `metadata_filter: dict` that narrows the operation to entries whose `metadata` matches. The filter uses the same MongoDB-style syntax as `db_find` — shorthand `{"field": value}` is equality, operators are explicit, dot-notation works for nested keys.

Supported operators: `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$exists`, `$regex`, `$contains` (array contains), `$elemMatch`, `$and`, `$or`, `$nor`, `$not`.

Operator semantics worth knowing (the filter syntax also applies to `db_find`):

- Multiple operators on one field AND together: `{"article_id": {"$exists": True, "$nin": dead_ids}}`.
- Negation operators (`$ne`, `$nin`) also match entries where the field is **missing** — add `"$exists": True` to restrict to entries that have the field. (The orphan-cleanup `run_id $ne` pattern relies on this: entries never stamped with a `run_id` also match.)
- `$in`/`$nin` compare values as text — keep ids as strings.
- `$in`/`$nin` lists accept at most 1,000 values.

```python
# Narrow semantic search to entries with specific metadata
await retrieval_query(
    query="status update",
    namespace="products",
    metadata_filter={"product_id": "X", "status": {"$ne": "archived"}},
)

# Compound conditions
await retrieval_query(
    query="incident timeline",
    metadata_filter={"$or": [{"priority": "high"}, {"score": {"$gte": 0.9}}]},
)
```

### `retrieval_delete` modes

The signature is `retrieval_delete(entry_id=None, namespace=None, metadata_filter=None)`. You must supply either `entry_id` or `namespace`. Three patterns:

1. **Single entry by id** — `entry_id` (optionally with `namespace` to disambiguate same-id-in-different-namespaces).
2. **Bulk by namespace + filter** — `namespace` + `metadata_filter`. Deletes every entry inside `namespace` (and its sub-namespaces) whose metadata matches. The canonical "orphan cleanup" pattern: re-ingest with the current `run_id`, then delete everything in scope where `run_id != current_run_id`. Sequencing matters: run the delete only after the re-ingest jobs have finished — entries still being indexed keep their previous `run_id` and would match the delete filter (see the async-indexing note above).
3. **Whole-subtree wipe** — `namespace` only (no `metadata_filter`). Deletes every entry under `namespace` and its sub-namespaces. Be careful — this is the bluntest tool here.

Rules:

- `metadata_filter` requires a `namespace`. There's no project-wide metadata-only delete.
- `metadata_filter` must be a dict; non-dict values are rejected.
- Wipe operations include sub-namespaces by default — `retrieval_delete(namespace="meetings")` also clears `meetings.q1`, `meetings.q2.standup`, etc.
- A successful delete returns `{"ok": true, "deleted_chunks": N}`. Entry IDs are namespace-local, so supply `namespace` when the same ID may exist more than once.

```python
# Orphan cleanup pattern keyed by the active ingestion run
await retrieval_delete(
    namespace="confluence",
    metadata_filter={
        "root_page_id": page_id,
        "run_id": {"$ne": current_run_id},
    },
)
```

## Database (schemaless document store)

Per-environment, MongoDB-like. Use it for state that needs to persist across runs.

```python
# Insert (single or batch); the collection is auto-created on first write
await db_insert("orders", {"order_id": "ORD-1", "amount": 99.0, "status": "pending"})
await db_insert("orders", [{"order_id": "ORD-2"}, {"order_id": "ORD-3"}])
# {"inserted": [{...full document plus system fields...}], "inserted_count": N}

# Find — note `fields` is a list of field paths, not a mongo-style projection map
result = await db_find(
    "orders",
    filter={"status": {"$in": ["pending", "shipped"]}, "amount": {"$gt": 50}},
    sort={"amount": -1},
    limit=10,                          # default 100, max 1000
    skip=0,
    fields=["customer", "amount"],     # include only these fields (list, not dict)
    distinct="customer",               # unique values of this field
)
# {"documents": [...], "count": N}
# When distinct is set: {"values": [...], "count": N}
# distinct ignores sort, limit, skip, and fields

# Update — sets fields on all matching docs; an empty filter updates every document
# Set a field to None to remove it.
await db_update("orders", filter={"order_id": "ORD-1"}, update={"status": "shipped"})
# {"updated_ids": [...], "updated_count": N}

# Upsert — update the first matching doc, or insert one if none match
await db_upsert(
    "orders",
    filter={"order_id": "ORD-1"},      # natural-key identifier
    update={"status": "shipped"},      # applied on both branches
    insert_only={"source": "etl"},     # applied only when inserting
)
# {"upserted_id": "<uuid>", "operation": "inserted" | "updated"}

# Delete (filter MUST be non-empty — no accidental drop-all)
await db_delete("orders", filter={"status": "cancelled"})
# {"deleted_ids": [...], "deleted_count": N}

# Count
await db_count("orders", filter={"status": "pending"})
# {"count": N}

# List collections with size info
await db_list_collections()
# {"collections": [{"name": "orders", "document_count": 100, "size_bytes": 50000}, ...], "total": 1}
```

System fields on every document (read-only): `_id` (UUID), `_created_at`, `_updated_at`.

Collection names are at most 50 characters, start with a lowercase letter, and contain only lowercase letters, digits, and underscores. `db_upsert` requires non-empty `filter` and `update` dicts and updates only the first match. On insert, it merges top-level equality fields from `filter`, then `update`, then `insert_only` (later values win). Operator, logical, and dotted filter fields are not copied into the inserted document. A `None` value in `update` removes the field on the update branch and is omitted from an inserted document.

### Filter operators and shorthand

Field-equality shorthand: `{"status": "pending"}` is the same as `{"status": {"$eq": "pending"}}`. Use it for simple matches.

Operators: `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$and`, `$or`, `$not`, `$nor`, `$exists`, `$contains`, `$elemMatch`, `$regex`, `$near`. `$regex` matching is always case-insensitive.

Nested fields use dot notation: `{"address.city": "Berlin"}`.

```python
filter = {
    "$and": [
        {"status": {"$in": ["pending", "shipped"]}},
        {"$or": [{"amount": {"$gt": 100}}, {"priority": "high"}]},
        {"tags": {"$contains": "rush"}},
        {"address.country": "DE"},
    ]
}
```

### Distance search with `$near`

Store coordinates as a `{"lat": number, "lng": number}` object. `$near` matches documents within `maxDistance` meters (optional) by great-circle distance. `db_find` returns them nearest first unless `sort` is set, and adds `_distance` in meters to each document. Use one `$near` per filter, at the top level (not inside `$or`, `$not`, or `$elemMatch`). Documents without numeric `lat`/`lng` never match. In `db_count`, `db_update`, `db_upsert`, and `db_delete` it only filters.

```python
await db_insert("workshops", {
    "name": "Garching",
    "services": ["tires", "inspection"],
    "location": {"lat": 48.2489, "lng": 11.6532},
})

result = await db_find("workshops", filter={
    "services": {"$contains": "tires"},
    "location": {"$near": {"lat": 48.137, "lng": 11.575, "maxDistance": 25000}},
}, limit=10)
# {"documents": [{"name": "Garching", ..., "_distance": 13726.69...}, ...], "count": N}
```

## Files

Project files: one folder tree per environment that keeps files between runs. Agents choose the path when they save; folders are just path prefixes and appear once a file lands in them.

- `file_list(folder="", recursive=False, filter=None, scope="all", sort=None, limit=100, skip=0)` returns `{"folder", "folders", "files", "count"}`. `folders` lists direct subfolders when not recursive. `scope="session"` or `"run"` keeps only files written in the current session or run.
- `file_read(path)` gives the model `{"file", "content"}` with the full text of text files, or the metadata plus an attachment for PDFs and images, up to 20 MiB. Other formats and larger files return an error; pass their path to `web_browser_upload` or a custom tool. In custom tools, `connic.tools.file_read` always returns `{"file": {...}, "content": bytes}`.
- `file_write(path, content, mime_type=None, metadata=None, overwrite=False)` writes a text file of up to 16 MiB and creates missing folders. Without `overwrite`, a taken path stores the new file as `name (2).ext`; the result's `_path` is the real path.
- `file_update(path, new_path=None, metadata=None)` moves or renames a file or a whole folder, and merges metadata into a file (`None` removes a field). Moving onto a taken path fails.
- `file_delete(path, recursive=False)` deletes a file, or a folder with `recursive=True`.

Every file carries system fields (`_path`, `_folder`, `_name`, `_mime_type`, `_size`, `_source`, `_agent`, `_run_id`, `_session_id`, `_id`, `_created_at`, `_updated_at`) plus custom metadata. Filters use the `db_find` operators on both. Custom metadata keys must not start with `_`.

```python
from connic.tools import file_list, file_update

result = await file_list("invoices", recursive=True, filter={"status": "new", "_mime_type": "application/pdf"})
for invoice in result["files"]:
    await file_update(invoice["_path"], metadata={"status": "processing"})
```

Files are up to 1 GiB through the dashboard and REST API, 100 MiB through the browser tools. Restrict an agent with the `files:` block in agent YAML (see agent-yaml reference).

## Web

`max_results` is capped at 10.

Each successful `web_search` costs €0.015. `web_read_page` costs €0.015 per extracted page, including each extracted PDF page.

`web_read_page` does not support `x.com`, `twitter.com`, or their subdomains. These requests return an `error` and are not charged.

```python
await web_search(query="connic.co pricing", max_results=5, country=None, include_news=False)
# {"results": [{"title": "...", "url": "...", "content": "..."}]}

await web_read_page(url="https://example.com", follow_redirects=True)
# {"markdown": "...", "url": "..."}  # url = the actually-scraped page (final URL after redirects)
# With follow_redirects=False, a redirect returns an error instead of the target page:
# {"markdown": None, "error": "URL redirected. Set follow_redirects=true to follow the redirect.", "redirect_url": "https://..."}
```

`follow_redirects` defaults to `True`. Pass `False` when you only want to read the exact URL you asked for — the call errors and returns the target in `redirect_url`.

### Browser control

Use `web_browser_*` in the agent's `tools:` list to enable browser control:

```yaml
tools:
  - web_browser_*
```

This enables `web_browser_open`, `web_browser_observe`, `web_browser_read`, `web_browser_act`, `web_browser_close`, `web_browser_screenshot`, `web_browser_mouse`, `web_browser_tabs`, `web_browser_dialog`, `web_browser_eval`, `web_browser_console`, `web_browser_upload`, `web_browser_download`, and `web_browser_pdf`. The agent calls these tools directly. Custom Python tools can import the individual functions from `connic.tools`.

```python
# tools/browser.py
from connic.tools import web_browser_open, web_browser_observe, web_browser_close

await web_browser_open("https://example.com")
# {"status": "active", "url": "...", "snapshot": "..."}
await web_browser_observe()
await web_browser_close()
```

Each run uses one browser, selected automatically for every browser tool call. Calling `web_browser_open(url)` again returns `{"status": "active", "message": "Browser already open"}` without navigating. Use `web_browser_tabs(action="new", url="...")` for another page.

`web_browser_observe(interactive_only=False, selector=None, include_urls=False)` reads the current page. Snapshots are cut off at 32000 characters, so narrow long pages with `interactive_only=True` (only elements the agent can act on) or a CSS `selector` such as `main`. `include_urls=True` adds link URLs. `web_browser_read()` returns the current page as Markdown in `content`, using the rendered page and its login state; content over 32000 characters sets `content_truncated`.

`web_browser_act(action, target=None, value=None)` performs an action and returns a new snapshot. Use element references from the latest snapshot, because navigation or page updates can change them.

| Action | `target` | `value` |
| --- | --- | --- |
| `click`, `double_click`, `hover` | Current element reference, such as `@e1` | Omit |
| `check`, `uncheck` | Current checkbox or radio reference | Omit |
| `scroll_into_view` | Current element reference | Omit |
| `fill` | Current input reference | Text to replace the field contents |
| `type` | Optional current element reference; omit for the focused field | Text to insert |
| `select` | Current select reference | Option value |
| `press` | Optional current element reference | Key, such as `Enter` |
| `keydown`, `keyup` | Optional current element reference | Key to hold or release, such as `Shift` |
| `scroll` | Omit | `up`, `down`, `left`, or `right` |
| `navigate` | Omit | HTTP or HTTPS URL |
| `back`, `forward`, `reload` | Omit | Omit |
| `wait` | Omit | Milliseconds from `100` to `10000` |
| `wait_for_text` | Omit | Text to appear on the page |
| `wait_for_url` | Omit | URL or pattern, such as `**/dashboard` |
| `wait_for_selector` | Omit | CSS selector to appear on the page |
| `wait_for_network_idle` | Omit | Omit |

The `wait_for_*` actions give up after 10 seconds and return an error with the current page.

`web_browser_screenshot(full_page=False)` returns a PNG attachment for the model to inspect. `full_page=True` captures the whole scrollable page; use viewport screenshots for mouse coordinates. `web_browser_mouse(action, ...)` controls the pointer using viewport coordinates. `web_browser_tabs(action="list")` returns tab IDs for switching or closing tabs. `web_browser_dialog(action, text=None)` accepts or dismisses a pending JavaScript dialog.

`web_browser_eval(script)` runs a JavaScript expression of up to 16384 characters in the active tab and returns its JSON value in `result`, along with a fresh snapshot. Promises are awaited; wrap statements that use `return` or `await` in `(async () => { ... })()`. Results over 32000 characters are cut off and set `result_truncated: true`; non-string values then come back as JSON text. A thrown error returns `error_code: "script_error"` with the message in `error`.

```python
from connic.tools import web_browser_eval

await web_browser_eval("[...document.querySelectorAll('h2')].map(h => h.textContent.trim())")
# {"status": "active", "result": ["Pricing", "FAQ"], "url": "...", "snapshot": "..."}
```

`web_browser_console()` returns `console_messages` (`type` and `text`, the 50 most recent) and `page_errors` (`text`, `line`, and `column`, the 20 most recent). Entries accumulate while the browser is open, newest last; `console_truncated` means older entries or long text were cut off.

`web_browser_download(target, include_content, save_to=None)` returns `download_id`, `name`, `mime_type`, and `size`. Set `include_content=True` to attach the file for the agent to read, or `False` for metadata only. `web_browser_pdf(include_content, save_to=None)` saves the current page as a PDF and returns the same fields. `web_browser_upload(target, download_id=None, path=None)` selects a downloaded file or saved PDF (`download_id`), or a file from project files (`path`); pass exactly one. Downloads are limited to 25 MiB and remain available until the browser closes. With `save_to`, the file is also stored in project files (see Files below), up to 100 MiB, and the result includes `saved_path`; a `save_to` ending in `/` is a folder that keeps the downloaded name. Files over 25 MiB are saved without an attachment.

The browser closes when its run ends. With sessions enabled, `session.browser` defaults to `true`: runs in the same connic session reuse saved cookies and local storage. `session: true` shares one session across the agent’s runs. Set `session.history: false` to retain the browser profile without conversation history. Runs without a session or with `session.browser: false` use a fresh profile. See [pricing](https://connic.co/pricing) for browser usage rates and environment limits.

Errors are returned in `error`. If an action returns `outcome_unknown`, observe the browser before deciding whether to act again.

## When to use predefined tools vs custom wrappers

Predefined tools are generic. For most projects you'll want thin custom wrappers that bake in project-specific defaults (namespace, collection, filter conventions). This makes the LLM's job easier — it sees `find_pending_orders()` instead of having to construct `$in` filters.

```python
# tools/orders.py
from connic.tools import db_find, db_insert

async def find_pending_orders(limit: int = 10) -> list:
    """List orders with status=pending, newest first.

    Args:
        limit: Max orders to return.

    Returns:
        List of order dicts.
    """
    result = await db_find(
        "orders",
        filter={"status": "pending"},
        sort={"_created_at": -1},
        limit=limit,
        fields=["order_id", "customer", "amount", "_created_at"],
    )
    return result["documents"]


async def save_order(order_id: str, amount: float) -> dict:
    """Persist a new pending order.

    Args:
        order_id: Unique order identifier.
        amount: Total in USD.

    Returns:
        The inserted document.
    """
    return await db_insert("orders", {
        "order_id": order_id,
        "amount": amount,
        "status": "pending",
    })
```

Then reference `orders.find_pending_orders` and `orders.save_order` from the agent. Skip `db_find` in the agent's `tools:` list entirely if you don't want the LLM forming raw queries.
