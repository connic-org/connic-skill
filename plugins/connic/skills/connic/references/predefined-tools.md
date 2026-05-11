# Predefined tools

Built-in tools that ship with the SDK. Reference them in an agent's `tools:` list by bare name — no module prefix.

```yaml
tools:
  - query_knowledge
  - trigger_agent
  - db_find
  - web_search
```

You can also call them from your own Python tools by importing from `connic.tools`:

```python
from connic.tools import query_knowledge, db_find, trigger_agent, web_search
```

## Agent orchestration

### `trigger_agent(agent_name, payload, wait_for_response=True, timeout_seconds=60)`

Run another agent from within this one.

```python
result = await trigger_agent(
    "billing-assistant",
    payload={"invoice_id": "INV-123"},
    wait_for_response=True,
    timeout_seconds=60,
)
# {"run_id": "...", "status": "completed"|"failed"|"timeout", "response": ...}
```

When `wait_for_response=False`, it returns immediately with `status: "started"`.

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
        "fields": {"customer_id": "cus_123"},             # optional structured data
    },
    wait_for_response=True,
)
```

Shape rules:

- `files` is a list. Each entry needs `data` (base64-encoded bytes as an ASCII string), `mime_type`, and `name`.
- `message` (or `text` as a fallback) becomes the leading text part the receiving LLM sees. If neither is set and `files` is present, the runtime falls back to `"Please analyze the attached file(s)."`.
- `fields` is optional structured context. When there's no `message`/`text`, `fields` is JSON-serialised into the text part.

Base64 inflates the payload by ~33% and the whole thing travels as a JSON string through the trigger endpoint. For anything larger than a few MB, prefer a reference — a presigned S3 URL, a knowledge-base `entry_id`, a project-database doc ID — and let the downstream agent fetch the binary via a tool.

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

## Knowledge base (vector store)

Per-environment vector store, optionally divided into namespaces. Namespaces are **dot-separated** (e.g. `policies.hr.leave`, `products.pricing`), max depth 10. Don't use slashes — they aren't valid namespace characters.

```python
# Semantic search
result = await query_knowledge(
    query="refund policy",
    namespace="policies",        # optional; omit for the default namespace
    min_score=0.7,               # default 0.7
    max_results=3,               # default 3
    metadata_filter={"source": "handbook"},  # optional; see "Metadata filters" below
)
# {"results": [{"score": 0.95, "content": "...", "metadata": {...}, "entry_id": "..."}, ...]}

# Insert or update — indexing is async
await store_knowledge(
    content="Refunds are issued within 7 days.",
    entry_id="refund-policy",    # optional; auto-generated if omitted
    namespace="policies",
    metadata={"source": "handbook"},
)
# {"entry_id": "...", "job_id": "...", "status": "pending", "queued": true, "success": true}

# Delete — three modes (see below)
await delete_knowledge(entry_id="refund-policy", namespace="policies")            # single entry
await delete_knowledge(namespace="confluence",                                    # bulk by metadata
                       metadata_filter={"root_page_id": page_id,
                                        "run_id": {"$ne": current_run_id}})
await delete_knowledge(namespace="meetings.q1")                                   # wipe a subtree

# List namespaces — default depth=1; depth=0 returns all descendants (max depth 10)
await kb_list_namespaces(parent=None, depth=1)
```

Knowledge entries become searchable only after the async indexing job finishes — don't query immediately after `store_knowledge`.

### Metadata filters on `query_knowledge` and `delete_knowledge`

Both tools accept an optional `metadata_filter: dict` that narrows the operation to entries whose `metadata` matches. The filter uses the same MongoDB-style syntax as `db_find` — shorthand `{"field": value}` is equality, operators are explicit, dot-notation works for nested keys.

Supported operators: `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$exists`, `$regex`, `$contains` (array contains), `$elemMatch`, `$and`, `$or`, `$nor`, `$not`.

```python
# Narrow semantic search to entries with specific metadata
await query_knowledge(
    query="status update",
    namespace="products",
    metadata_filter={"product_id": "X", "status": {"$ne": "archived"}},
)

# Compound conditions
await query_knowledge(
    query="incident timeline",
    metadata_filter={"$or": [{"priority": "high"}, {"score": {"$gte": 0.9}}]},
)
```

### `delete_knowledge` modes

The signature is now `delete_knowledge(entry_id=None, namespace=None, metadata_filter=None)`. You must supply either `entry_id` or `namespace`. Three patterns:

1. **Single entry by id** — `entry_id` (optionally with `namespace` to disambiguate same-id-in-different-namespaces). The legacy shape.
2. **Bulk by namespace + filter** — `namespace` + `metadata_filter`. Deletes every entry inside `namespace` (and its sub-namespaces) whose metadata matches. The canonical "orphan cleanup" pattern: re-ingest with the current `run_id`, then delete everything in scope where `run_id != current_run_id`.
3. **Whole-subtree wipe** — `namespace` only (no `metadata_filter`). Deletes every entry under `namespace` and its sub-namespaces. Be careful — this is the bluntest tool here.

Rules:

- `metadata_filter` requires a `namespace`. There's no project-wide metadata-only delete.
- `metadata_filter` must be a dict; non-dict values are rejected.
- Wipe operations include sub-namespaces by default — `delete_knowledge(namespace="meetings")` also clears `meetings.q1`, `meetings.q2.standup`, etc.

```python
# Orphan cleanup pattern — re-ingested with run_id = current_run_id, now drop stale entries
await delete_knowledge(
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
# Insert (single or batch)
await db_insert("orders", {"order_id": "ORD-1", "amount": 99.0, "status": "pending"})
await db_insert("orders", [{"order_id": "ORD-2"}, {"order_id": "ORD-3"}])

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

# Update — sets fields on all matching docs; set a field to None to remove it
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

# Count
await db_count("orders", filter={"status": "pending"})

# List collections with size info
await db_list_collections()
# [{"name": "orders", "document_count": 100, "size_bytes": 50000}, ...]
```

System fields on every document (read-only): `_id` (UUID), `_created_at`, `_updated_at`.

### Filter operators and shorthand

Field-equality shorthand: `{"status": "pending"}` is the same as `{"status": {"$eq": "pending"}}`. Use it for simple matches.

Operators: `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`, `$and`, `$or`, `$not`, `$nor`, `$exists`, `$contains`, `$elemMatch`, `$regex`. For case-insensitive regex, use the inline flag: `{"$regex": "(?i)foo"}`.

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

```python
await web_search(query="connic.co pricing", max_results=5, country=None, include_news=False)
# {"results": [{"title": "...", "url": "...", "content": "..."}]}

await web_read_page(url="https://example.com", follow_redirects=True)
# {"markdown": "...", "url": "..."}
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
