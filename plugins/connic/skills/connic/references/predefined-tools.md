# Predefined tools

Built-in tools that ship with the SDK. Reference them in an agent's `tools:` list by bare name — no module prefix.

```yaml
tools:
  - retrieval_query
  - trigger_agent
  - db_find
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
# {"run_id": "...", "status": "completed"|"failed"|"cancelled"|"awaiting_approval"|"timeout", "response": ..., "error": ...?}
```

Pass a dict or list for structured JSON. Strings can contain plain text or encoded JSON.

When `wait_for_response=False`, it returns immediately with just `run_id` (no `status` or `response`).

**Inside the deploy-gate test container**, `trigger_agent` runs the child agent in-process (instead of hitting the live deployment) so tests can assert on the chain via `expected_child_agents` in `tests/*.yaml`. See [cli-and-dev.md](cli-and-dev.md#asserting-on-triggered-agents). Production behaviour is unchanged.

#### Passing files to the triggered agent

`trigger_agent` has no `files` parameter, but the receiving agent's runtime recognises a specific dict shape in the payload and reconstructs binary file parts from it — the same shape that inbound multipart webhooks and Telegram media use, so it's stable and supported:

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

Base64 inflates the payload by ~33% and the whole thing travels as a JSON string through the trigger endpoint. For anything larger than a few MB, prefer a reference — a presigned S3 URL, a Retrieval `entry_id`, a project-database doc ID — and let the downstream agent fetch the binary via a tool.

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

Managed semantic retrieval for each environment, optionally divided into namespaces. Namespaces are **dot-separated** (e.g. `policies.hr.leave`, `products.pricing`), max depth 10. Don't use slashes — they aren't valid namespace characters. Namespace scoping always covers the whole subtree: querying `policies` also searches `policies.hr.leave` — the same inclusion rule deletes use.

```python
# Semantic search
result = await retrieval_query(
    query="refund policy",
    namespace="policies",        # optional; omit for the default namespace
    min_score=0.3,               # default 0.3
    max_results=3,               # default 3
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

Retrieval entries become searchable only after the async indexing job finishes — don't query immediately after `retrieval_store`. The same applies to metadata-filter deletes: a delete fired while ingestion jobs are pending only sees already-indexed entries, and re-ingested entries keep their *previous* metadata until their job commits. Re-storing an existing `entry_id` + namespace atomically replaces that entry's content when the job lands — there's never mixed old/new state per entry, but there's no in-agent way to poll a `job_id`, so schedule dependent cleanup as a later run (e.g. via `trigger_agent_at`) rather than inline.

### Metadata filters on `retrieval_query` and `retrieval_delete`

Both tools accept an optional `metadata_filter: dict` that narrows the operation to entries whose `metadata` matches. The filter uses the same MongoDB-style syntax as `db_find` — shorthand `{"field": value}` is equality, operators are explicit, dot-notation works for nested keys.

Supported operators: `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$exists`, `$regex`, `$contains` (array contains), `$elemMatch`, `$and`, `$or`, `$nor`, `$not`.

Operator semantics worth knowing (same engine as `db_find`):

- Multiple operators on one field AND together: `{"article_id": {"$exists": True, "$nin": dead_ids}}`.
- Negation operators (`$ne`, `$nin`) also match entries where the field is **missing** — add `"$exists": True` to restrict to entries that have the field. (The orphan-cleanup `run_id $ne` pattern relies on this: entries never stamped with a `run_id` also match.)
- `$in`/`$nin` compare values as text — keep ids as strings.
- No length limit on `$in`/`$nin` lists; the list binds as one array parameter, so thousands of ids are fine.

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

1. **Single entry by id** — `entry_id` (optionally with `namespace` to disambiguate same-id-in-different-namespaces). The legacy shape.
2. **Bulk by namespace + filter** — `namespace` + `metadata_filter`. Deletes every entry inside `namespace` (and its sub-namespaces) whose metadata matches. The canonical "orphan cleanup" pattern: re-ingest with the current `run_id`, then delete everything in scope where `run_id != current_run_id`. Sequencing matters: run the delete only after the re-ingest jobs have finished — entries still being indexed keep their previous `run_id` and would match the delete filter (see the async-indexing note above).
3. **Whole-subtree wipe** — `namespace` only (no `metadata_filter`). Deletes every entry under `namespace` and its sub-namespaces. Be careful — this is the bluntest tool here.

Rules:

- `metadata_filter` requires a `namespace`. There's no project-wide metadata-only delete.
- `metadata_filter` must be a dict; non-dict values are rejected.
- Wipe operations include sub-namespaces by default — `retrieval_delete(namespace="meetings")` also clears `meetings.q1`, `meetings.q2.standup`, etc.
- A successful delete returns `{"ok": true, "deleted_chunks": N}`. Entry IDs are namespace-local, so supply `namespace` when the same ID may exist more than once.

```python
# Orphan cleanup pattern — re-ingested with run_id = current_run_id, now drop stale entries
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

Collection names are at most 50 characters, start with a lowercase letter, and contain only lowercase letters, digits, and underscores. `db_upsert` requires non-empty `filter` and `update` dicts and updates only the first match. On insert, it merges top-level equality fields from `filter`, then `update`, then `insert_only` (later values win). Operator, logical, and dotted filter fields are not copied into the inserted document. A `None` value removes a field only on the update branch; on insertion it is stored as JSON null.

### Filter operators and shorthand

Field-equality shorthand: `{"status": "pending"}` is the same as `{"status": {"$eq": "pending"}}`. Use it for simple matches.

Operators: `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$and`, `$or`, `$not`, `$nor`, `$exists`, `$contains`, `$elemMatch`, `$regex`. `$regex` matching is always case-insensitive.

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

## Web

`max_results` is capped at 10.

Every `web_search` or `web_read_page` call consumes one additional Project-credit run unit. A base agent run that makes two web-tool calls therefore counts as three run units.

```python
await web_search(query="connic.co pricing", max_results=5, country=None, include_news=False)
# {"results": [{"title": "...", "url": "...", "content": "..."}]}

await web_read_page(url="https://example.com", follow_redirects=True)
# {"markdown": "...", "url": "..."}  # url = the actually-scraped page (final URL after redirects)
# With follow_redirects=False, a redirect returns an error instead of the target page:
# {"markdown": None, "error": "URL redirected. Set follow_redirects=true to follow the redirect.", "redirect_url": "https://..."}
```

`follow_redirects` defaults to `True`. Pass `False` when you only want to read the exact URL you asked for — the call errors and returns the target in `redirect_url`.

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
