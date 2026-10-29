# Apps

Apps are static frontends declared in code. connic builds each one on every deploy, serves it on its own URL (`https://<key>-<8 hex characters>.connic.app`, plus an optional custom domain), signs visitors in, lets the browser call the app's connectors, and gives browser code read and write access to the environment's database, files, runs, sessions, and approvals, plus search in chosen Retrieval namespaces, under access rules declared next to it. Apps are available on pro and enterprise.

Apps are unrelated to **OAuth apps**, which hold provider settings for per-user MCP sign-in (see [guardrails-schemas-mcp.md](guardrails-schemas-mcp.md#per-user-sign-in-oauth-apps)).

## Files

```
apps/
├── portal.yaml            # declares the app; the file name is its key
└── portal/                # sources: package.json plus a lockfile
    ├── package.json
    ├── package-lock.json  # or pnpm-lock.yaml or yarn.lock
    └── src/
```

- One `apps/<key>.yaml` per app, directly in `apps/`, with the `.yaml` extension. The key starts with a letter or digit and uses letters, digits, `-` and `_` (100 characters max); it can't be set inside the file.
- `apps/<key>/` must exist with a `package.json` and a lockfile. A folder without a matching `.yaml` fails lint.
- Deploys upload the sources without `node_modules`, hidden files, and `dist`, `build`, `out`, and `coverage` folders at the top of each app. Allowed source types: `.js .mjs .cjs .jsx .ts .mts .cts .tsx .vue .svelte .astro .html .htm .css .scss .sass .less .json .jsonc .md .mdx .txt .yaml .yml .toml .svg .xml .webmanifest .graphql .gql .lock` (UTF-8 text) and `.png .jpg .jpeg .gif .webp .avif .ico .woff .woff2 .ttf .otf .mp3 .mp4 .webm .pdf`. Anything else, including files without an extension, fails with `File type '...' not allowed in apps/`.

## `apps/<key>.yaml`

```yaml
# apps/portal.yaml
name: Support portal                # optional, defaults to the key
build:
  command: npm run build            # runs in apps/portal/
  output: dist                      # folder with index.html, relative to apps/portal/
  install: npm ci                   # optional, defaults to the lockfile's install command
  node: "24"                        # "22" or "24" (default)
auth:                               # optional; omit to manage sign-in in the dashboard
  mode: oidc                        # public, oidc or connic
  issuer: https://login.example.com
  client_id: ${PORTAL_CLIENT_ID}
  client_secret: ${PORTAL_CLIENT_SECRET}
  allowed_domains: [example.com]    # optional email domains
  allowed_groups: [support]         # optional, matched against groups_claim
connectors: [support-chat]          # apps_only connectors the browser may call
domain: portal.example.com          # optional active custom domain of the project
lookups:                            # documents resolved once per request
  member:
    collection: members
    where: doc.email == user.email
access:                             # what the browser may read and change; the rest is denied
  database:
    tickets:
      read: doc.author_id == user.id or member.role == "agent"
      insert: user.signed_in == true
      update: doc.author_id == user.id
      set:
        author_id: user.id
  files:
    reports/:
      read: "'support' in user.groups"
  runs:
    read: run.context.end_user.id == user.id
  retrieval:                        # Retrieval namespaces the browser may search
    help:
      search: true
environments:                       # omit to deploy to every environment
  Staging:
    auth:
      mode: connic
      groups: [Admin]
  Production: {}
```

| Field | Rules |
| --- | --- |
| `name` | 1 to 200 characters, no `${VAR}`. |
| `build.command` | Required, one line, no `${VAR}`. |
| `build.output` | Required folder inside the app, such as `dist`. It must contain `index.html`. |
| `build.install` | Defaults to `npm ci`, `pnpm install --frozen-lockfile`, or `yarn install --frozen-lockfile` by lockfile. |
| `build.node` | `"22"` or `"24"`, default `"24"`. |
| `auth` | See [Sign-in](#sign-in). |
| `connectors` | Keys of `connectors/*.yaml` files with `apps_only: true`. See [Calling agents](#calling-agents-through-connectors). |
| `domain` | Hostname or ID of an active custom domain of the project that no connector or other app uses. |
| `content_security_policy` | One line, up to 4,000 characters; replaces the default policy. |
| `lookups`, `access` | See [Access rules](#access-rules). |
| `environments` | Environment names, matched exactly; a name that doesn't exist fails the deploy. An override replaces the whole top-level value of `name`, `auth`, `connectors`, `domain`, `content_security_policy`, `lookups`, or `access`; `null` removes a top-level `domain`, `content_security_policy`, `lookups`, or `access`. `build` is the same everywhere. |

`${VAR}` works only in `domain` and in `auth.issuer`, `auth.client_id`, and `auth.client_secret`, resolved at deploy from the target environment's Variables; `client_secret` must be a variable.

## Building

- Static output only. Vite (React, Vue, Svelte, Solid), Angular, Astro, Next.js with `output: "export"`, SvelteKit with adapter-static, and Nuxt `generate` work. Server rendering, API routes, framework middleware, and on-request image optimization don't run: put that logic in an agent behind an `apps_only` connector (a `type: tool` agent answers without a model call).
- A deploy installs with the install command in a clean Node environment, runs `build.command`, checks for `<output>/index.html`, and fails with the build output in the deployment log otherwise. A rollback serves the older build again.
- Builds get no Variables. The app reads its settings at runtime from `/_connic/config.json`, so one build serves every environment.
- Paths without an extension try `<path>.html`, `<path>/index.html`, then `index.html`, so client-side routing works. A missing file with an extension gets the build's `404.html` when it has one.
- Default `Content-Security-Policy`: `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self' data:; connect-src 'self'; frame-ancestors 'self'; base-uri 'self'; form-action 'self'; object-src 'none'`. Calls to other origins, third-party scripts, or frames need `content_security_policy`; Next.js static exports need `script-src 'self' 'unsafe-inline'`.

## Sign-in

| Mode | Who can open the app | `user.id` |
| --- | --- | --- |
| `public` | Anyone with the link | An ID connic derives per app from a secret visitor cookie |
| `oidc` | People your OpenID Connect provider signs in, optionally limited by `allowed_domains` (email domains) and `allowed_groups` (matched against the `groups_claim` claim, default `groups`) | The ID token's `sub` |
| `connic` | Members of the project, optionally limited to the permission groups in `groups`; the owner always passes | The connic user ID |

- `oidc` needs `issuer` (https), `client_id`, and `client_secret`. Register a web app at the provider with the redirect URI `https://api.connic.co/v1/apps/auth/callback` (shown as `sign_in.redirect_uri` in the dashboard and connic MCP `get_app`). `openid`, `profile`, and `email` are always requested; `scopes` adds more. An ID token with `email_verified: false` gives the visitor no email, so `allowed_domains` turns them away. In the dashboard and connic MCP, changing `issuer` or `client_id` needs the client secret again.
- `connic` suits internal tools and staging apps. Customers' users belong on `oidc` or `public`; don't make them project members.
- **Where it's set, per setting (sign-in and custom domain):** declared in `apps/<key>.yaml`, code wins and the dashboard and connic MCP can't change it (409). Not declared, it's managed under **Apps → Details → Settings** in the dashboard or with connic MCP `update_app` (permission `apps.manage`). A new app with no `auth` anywhere uses `connic` for every project member. When a file stops declaring a setting, its last value stays and the dashboard takes it over, so a deploy never silently changes who can open an app.
- Changing sign-in ends the app's sessions. Sessions last up to 12 hours; connic renews the short-lived sign-in token on the visitor's requests and checks access again each time (project membership for `connic`, the domain and group rules for `oidc`).
- Without a session, page navigations redirect to sign-in and every other request gets 401 `{"error": "Sign-in required"}`. On 401, send the visitor to `/_connic/login?return=<path>`.

Reserved paths on every app host:

| Path | What it does |
| --- | --- |
| `/_connic/config.json` | `{app: {key, name}, environment, auth, connectors, login_url, logout_url}`; the URLs are `null` on public apps |
| `/_connic/me` | The visitor: `{user: {id, email, name, groups, signed_in}, app: {key, name}}` |
| `/_connic/login?return=/path`, `/_connic/logout` | Start sign-in, sign out |
| `/_connic/c/<connector key>` | The app's connectors |
| `/_connic/db/...`, `/_connic/files/...`, `/_connic/retrieval/search`, `/_connic/runs...`, `/_connic/sessions/<id>`, `/_connic/approvals...` | [Data endpoints](#data-endpoints) |

## Calling agents through connectors

```yaml
# connectors/support-chat.yaml
type: webhook
mode: sync            # apps call sync webhook or websocket connectors
apps_only: true
agents: [support-assistant]
```

```js
const response = await fetch("/_connic/c/support-chat", {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify({ message: "Where is my order?" }),
});
const { result } = await response.json(); // the sync webhook response
```

- `apps_only: true` is a top-level connector field for sync `webhook` and `websocket` connectors. Such a connector accepts only requests connic forwards from the apps that list it; its secret and **Require Authentication** no longer apply.
- Calls must come from the app's own pages: a request with another `Origin` or a cross-site `Sec-Fetch-Site` gets 403 `{"error": "Only same-origin requests are allowed"}`, and a WebSocket handshake must carry the app's `Origin`.
- A connector listed by an app must be deployed to every environment the app is; lint reports a mismatch.
- WebSocket: ``new WebSocket(`wss://${location.host}/_connic/c/support-chat`)``, with the [websocket message protocol](connectors.md#websocket). An optional last segment, `/_connic/c/<key>/<connection id>`, allows one active connection per ID.
- The connector's `rate_limit` works as on its regular URL, and `per_ip` sees the visitor's IP.
- Runs started through an app get `context["app"]` (`{"id", "key"}`) and `context["end_user"]` (`{"id", "email", "name", "groups", "signed_in"}`), set by connic from the forwarded request, never from the payload. On public apps `end_user` is the anonymous visitor: the per-app `id` from the visitor cookie, `email` and `name` null, `signed_in` false.

```python
# middleware/support-assistant.py
from connic import StopProcessing


async def before(content, context):
    user = context.get("end_user") or {}
    if not user.get("signed_in"):
        raise StopProcessing("Sign in to use the assistant.")
    context["customer_email"] = user["email"]
    return content
```

`session: {key: context.end_user.id}` keeps one conversation per visitor, and `oauth.user: context.end_user.id` in `mcp_servers` uses the visitor's own OAuth token for an MCP server.

## Access rules

`access` lists, per resource, the operations the browser may perform. A missing resource or operation is denied. Each rule is an expression string; YAML `true` and `false` become `"true"` and `"false"`.

| Resource | Operations | Record name |
| --- | --- | --- |
| `database.<collection>` | `read`, `insert`, `update`, `delete`, plus `set` | `doc`: stored fields plus `doc.id`, `doc.created_at`, `doc.updated_at` |
| `files.<folder>` | `read`, `write`, `update`, `delete` | `file`: `path`, `folder`, `name`, `size`, `mime_type`, `metadata.<key>`, `created_at`, `updated_at` |
| `runs` | `read`, `cancel` | `run`: `id`, `agent`, `status`, `context`, `input`, `output`, `queued_at`, `ended_at`, `duration_ms` |
| `sessions` | `read`, `delete` | `session`: `id`, `agent`, `state`, `created_at`, `updated_at` |
| `approvals` | `read`, `decide` | `approval`: `id`, `agent`, `tool`, `params`, `status`, `context` (its run's), `created_at` |
| `retrieval.<namespace>` | `search` | none: the rule decides for the whole namespace |

Every rule may use `user` (`id`, `email`, `name`, `groups`, `claims`, `signed_in`) and the lookup names. Database `insert` and `update` and files `write` and `update` may also use `new`: on insert each document the browser sends, on update only the fields in `update`, on upload `{path, size, mime_type, metadata}`, on file update the file after the change. `set` values may use `user` and lookups only. `user.groups` holds the OIDC groups claim or the connic permission group names; `user.claims` holds the OIDC ID token claims (`{}` otherwise).

Grammar: comparisons (`==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `not in`, and `is`, `is not` with `null`, `true`, or `false` only), `and`, `or`, `not`, parentheses, dot and subscript paths, and string, number, boolean, null, list, and object literals. Comparisons take values, not other conditions. No calls, arithmetic, unary minus, or `_`-prefixed attributes; 2,000 characters max.

How rules apply:

- `read`, `delete`, and the stored-record part of `update`, `cancel`, and `decide` are **filters**. connic evaluates the parts that don't use the record per request and turns the rest into a query filter ANDed with the browser's filter, so results, counts, and paging only ever include allowed records. A rule that is false for every record returns 403 `Access denied`; a single record the rule excludes returns 404.
- `insert` and files `write` are checked against `new` before the write. `set` fields are written by connic on insert, overriding what the browser sent; if one can't be evaluated the insert is denied. An update that names a `set` field returns 400.
- **Three-valued logic.** A comparison whose side is missing or null (a missing field, a null `user.email`, a lookup that found nothing) is unknown. `not` keeps it unknown, `and` and `or` combine it the usual way (`false and unknown` is false, `true or unknown` is true), and a rule that ends unknown denies. Only a comparison with the literal `null` tests for null: `doc.x == null` holds when `x` exists and is null, and `doc.x != "v"` doesn't match documents where `x` is null or missing.
- **Typed values.** Each JSON type equals only its own (`true` isn't `1`, `"1"` isn't `1`); `<`, `<=`, `>`, `>=` compare two numbers or two texts. `x in [...]` compares typed list items, `x in <object>` checks its keys, and `x in <text>` is exact equality, not a substring test (`"x" in doc.tags` matches a tags array or object key). Timestamps compare as instants, written as ISO 8601 strings. `doc.id`, `run.id`, `approval.id`, and `status` take only `==`, `!=`, `in`, `not in`.
- **Size.** A rule's database or files filter must stay within depth 10, 100 branches per `and`/`or`, 256 terms, and 1,000 values per `in`. The deploy rejects rules that exceed it; if values known only per request (such as a long `user.groups`) push a filter over, the request fails with 500 and a message naming the limit.
- Files: folder keys are normalized without slashes (`reports/` is `reports`). A rule covers its folder and everything below; the most specific folder wins. Moving a file also needs the target folder's `write` rule. Uploads never overwrite: a taken path gets a numbered name, and the completed file's `_path` is the real one.
- File listing is one level deep and shows only subfolders that hold files the visitor can read under their own folder's rule. A folder without a rule of its own, such as the root, can be listed when a rule folder below it is readable; it returns those subfolders with `files: []`. With no readable rule folder at or below it, listing returns 403.
- Retrieval: a `search` rule covers its namespace and every namespace below it, since a search returns passages from all of them, and the visitor can read the content and metadata of every entry there. A listed namespace can't sit inside another listed one (`help` and `help.internal` fail the deploy); keep content for different audiences in separate namespaces such as `public.help` and `internal.help`. Entries without a namespace can't be searched from an app. Retrieval search stays off the REST API and connic MCP, so apps and agents (`retrieval_query`) are the only ways to search outside the dashboard.

**Lookups** resolve once per request, only when a rule uses them: the oldest document in `collection` matching `where` (which may use `doc` and `user` only), as its fields plus `id`, `created_at`, `updated_at`, or `null` when nothing matches. Every comparison with a null lookup is unknown, so it denies even under `not`. Names must be identifiers other than `user`, `doc`, `new`, `file`, `run`, `session`, `approval`, `true`, `false`, `null`.

**Public visitors.** On a public app, `user` is `{id, email: null, name: null, groups: [], claims: {}, signed_in: false}`. The `id` is derived per app from a secret visitor cookie, so knowing it doesn't let anyone act as that visitor, but clearing cookies or switching browsers gives a new ID and loses access to records tied to the old one. Rules on `user.email` or `user.name` never match a public visitor, not even documents whose field is null. Sign-in can be switched to public in the dashboard, so state the intent explicitly where it matters: `user.signed_in == true and doc.author_id == user.id`.

**Checks and errors.** `connic lint` checks syntax and which names each rule may use. The deploy (and every `connic dev` upload) also compiles each rule and fails with `apps/<key>.yaml: <yaml path>: <reason>`:

| Error | Fix |
| --- | --- |
| `'new' isn't available here; use doc, user` | Only insert, update, and files write/update see `new`. |
| `unknown name 'x'` | Declare the lookup or fix the name. |
| `unsupported expression element Call` (or `BinOp`, `USub`) | No calls or arithmetic; compare values directly. |
| `use is and is not only with null, true or false` | Use `==` or `!=` for other values. |
| `compare values, not conditions; combine comparisons with and, or and not` | Don't nest a comparison inside another, such as `(doc.a == 1) == true`. |
| `it compiles to a database filter that is too large: ...` | Shorten long `or` chains and `in` lists. |
| `private attribute '_id' isn't allowed` | Use `doc.id`, `doc.created_at`, `doc.updated_at`. |
| `can't compare two fields of doc` | Compare a record field with a value, `user`, or a lookup. |
| `can't compare fields of doc with new` | Check `new` and `doc` in separate comparisons. |
| `access.retrieval.help.internal: help already covers it; declare only one of them` | Move the content into a namespace outside `help`, or drop one of the two. |
| `compare doc fields with a value, e.g. doc.done == true` | No bare record fields. |
| `doc.id can only be compared with ==, !=, in and not in` | No ordering on ids or `status`. |
| `file has no field x; use path, folder, ...` / `compare file.metadata fields, e.g. file.metadata.owner` | Custom file fields live under `file.metadata.<key>`. |
| `connector 'x' needs apps_only: true in connectors/x.yaml` | Set it, or remove the connector from `connectors`. |
| `auth.groups: permission group 'x' doesn't exist in this project` | Use an existing permission group name. |
| `Apps aren't enabled for this project's plan` | Apps need pro or enterprise. |

## Data endpoints

Browser code calls these paths on its own host. A request with another `Origin` or a cross-site `Sec-Fetch-Site` is refused. Writes (every method except GET and HEAD) also need `Origin` equal to the app's origin and `Content-Type: application/json`, including the bodyless `POST /_connic/runs/<id>/cancel` and `DELETE /_connic/sessions/<id>`; anything else gets 403 `{"error": "Only same-origin JSON requests are allowed"}`. Bodies are limited to 1 MB (413). A `fetch` from the page meets these rules; other sites and servers can't use the endpoints. Data requests never start runs: call a connector for that.

| Request | Body or query | Response |
| --- | --- | --- |
| `GET /_connic/me` | | `{user, app}` |
| `POST /_connic/db/<collection>/find` | `{filter?, sort?: {field: 1 \| -1}, limit? (default 25, max 100), skip?, fields?}` | `{documents, count, total}` |
| `POST /_connic/db/<collection>/count` | `{filter?}` | `{count}` |
| `POST /_connic/db/<collection>/insert` | `{document}` or `{documents}` (max 50) | 201 `{inserted, inserted_count}` |
| `POST /_connic/db/<collection>/update` | `{filter, update}` (non-empty filter; `null` removes a field) | `{updated}` |
| `POST /_connic/db/<collection>/delete` | `{filter}` (non-empty) | `{deleted}` |
| `POST /_connic/files/list` | `{folder?, filter?, limit?, cursor?}` | `{folder, folders, files, count, total, next_cursor}` |
| `POST /_connic/files/download` | `{path}` | `{url, expires_at}` |
| `POST /_connic/files/upload` | `{path, size, mime_type?, metadata?}` | 201 `{upload_id, path, size, part_bytes, part_count, upload_url, token, expires_at}` |
| `POST /_connic/files/update` | `{path, new_path?, metadata?}` (metadata merges; `null` removes a key) | the file |
| `POST /_connic/files/delete` | `{path}` | `{deleted}` |
| `POST /_connic/retrieval/search` | `{namespace, query (max 1,000 characters), limit? (default 5, max 20), min_score? (0 to 1, default 0.3)}` | `{results: [{entry_id, namespace, content, score, metadata}]}`, best match first |
| `GET /_connic/runs` | `?agent&status&limit&cursor` | `{runs, next_cursor}` |
| `GET /_connic/runs/stats` | `?agent&days` (1 to 90, default 7) | `{days, total, statuses, average_duration_ms, daily: [{day, runs}]}` |
| `GET /_connic/runs/<id>` | | run |
| `POST /_connic/runs/<id>/cancel` | | run |
| `GET /_connic/sessions/<id>` | | `{session: {id, agent, state, created_at, updated_at}, events}` |
| `DELETE /_connic/sessions/<id>` | | `{deleted: true}` |
| `GET /_connic/approvals` | `?status=pending\|approved\|rejected\|timeout` | `{approvals}` (up to 100, pending first) |
| `POST /_connic/approvals/<id>/decide` | `{approved, message?, response?}` | approval |

- A run is `{id, agent, status, input, output, error, context, queued_at, started_at, ended_at, duration_ms}`, with `input` and `output` as raw strings. An approval is `{id, run_id, agent, tool, params, message, response_label, status, created_at, resolved_at, reason}`; `decide` needs `response` when `response_label` is set.
- Filters use the [`db_find` operators](predefined-tools.md#filter-operators-and-shorthand). Collection system fields are `_id`, `_created_at`, `_updated_at`; file filters use `_path`, `_folder`, `_name`, `_size`, `_mime_type`, `_created_at`, `_updated_at`, and metadata keys at the top level.
- Errors are `{"error": "..."}`: 400 `<field>: <message>` (unknown body keys are rejected too), 401 `Sign-in required`, 403 `Access denied`, 404 for records the rules exclude, 429 `Too many requests. Try again in a minute.` Each visitor gets 300 reads, 60 writes, and 30 Retrieval searches per minute, each app 6,000, 1,200, and 600; public visitors are counted by IP. Debounce search-as-you-type.

```js
async function connic(path, body) {
  const response = await fetch(`/_connic/${path}`, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(body ?? {}),
  });
  if (response.status === 401) {
    location.href = `/_connic/login?return=${encodeURIComponent(location.pathname)}`;
  }
  if (!response.ok) throw new Error((await response.json()).error);
  return response.json();
}

const { documents } = await connic("db/tickets/find", { filter: { status: "open" }, sort: { _created_at: -1 } });
```

### File transfers

`upload_url` and the download `url` are same-origin paths under `/_connic/files/transfer/`. Requests from other sites are refused, writes need the app's `Origin`, and on sign-in apps every transfer needs a session.

1. `POST /_connic/files/upload` checks the folder's `write` rule and returns `upload_url` (`/_connic/files/transfer/uploads/<upload_id>`), `token`, `part_bytes`, and `part_count`.
2. Send each part with `PUT {upload_url}/parts/{n}?token=<token>`, `n` from 1, exactly `part_bytes` bytes except the last (at most 16 MiB, with a `Content-Length`: send a `Blob` slice, not a stream).
3. Finish with `POST {upload_url}/complete?token=<token>`, which returns the file. `DELETE {upload_url}?token=<token>` aborts.

```js
const upload = await connic("files/upload", { path: `reports/${file.name}`, size: file.size, mime_type: file.type || undefined });
const token = encodeURIComponent(upload.token);
for (let n = 1; n <= upload.part_count; n++) {
  const part = file.slice((n - 1) * upload.part_bytes, n * upload.part_bytes);
  await fetch(`${upload.upload_url}/parts/${n}?token=${token}`, { method: "PUT", body: part });
}
const stored = await (await fetch(`${upload.upload_url}/complete?token=${token}`, { method: "POST" })).json();
```

`POST /_connic/files/download` checks the `read` rule and returns `{url: "/_connic/files/transfer/download?token=...", expires_at}`, a short-lived link to GET (Range requests work), for example as an `href` or `src`.

## `connic dev`

- `connic dev` uploads `apps/*.yaml` and `connectors/*.yaml` with the other project files (not the app sources) and applies them to the session's test environment like a deploy: routes, sign-in, access rules, and connectors. Rule and connector errors show as `Upload failed: ...`.
- It builds each app locally by running `build.command` in `apps/<key>/` and uploads the output folder, then prints `<key> → <url>`. It doesn't run the install command: install dependencies in `apps/<key>/` first. Node.js must be on `PATH`.
- It watches `apps/<key>/` (ignoring `node_modules`, hidden files, the output folder, and the top-level build folders) and rebuilds when contents change, or when the app's `build` settings change. A failed build prints its output and keeps the last good build live.
- `/_connic/` paths only exist on the app's host, so test sign-in, connectors, and data against the printed URL rather than the framework's own dev server. Test environments serve apps uncached.

## Limits

| | pro | enterprise |
| --- | --- | --- |
| Apps per environment | 5 | 5 (custom plans can differ) |
| Build output per app | 25 MB | 25 MB |
| App build time per deploy | 10 minutes | 10 minutes |

Free and developer plans have no apps; a deploy that declares one fails. Build output is also limited to 20,000 files per deploy. App sources count toward the 25 MB deploy upload, with a 20 MB budget of their own. Custom domains need pro or enterprise. `connic dev` uploads have the same output limit.

## Bot protection on public apps

Anyone who opens a public app can call its connectors, and every call starts a run. Before putting an LLM agent behind a public app:

- Set the connector's `rate_limit` (`total` and `per_ip`; see [connectors.md](connectors.md)).
- Add a challenge widget from a bot-protection service to the page and send its token with each request, for example in an `x-challenge-token` header. Verify the token in the agent's middleware with the service's server-side verification API, keeping its secret in Variables, and stop the run when it fails.
- Add the service's script and frame origins to `content_security_policy`; the default policy only allows the app's own origin.
- Challenge tokens are usually single-use, so prefer a sync webhook connector with a fresh token per request; on a WebSocket every message is its own run.

```python
# middleware/support-assistant.py (requirements.txt: httpx)
import os

import httpx
from connic import StopProcessing


async def before(content, context):
    token = (context.get("request") or {}).get("headers", {}).get("x-challenge-token")
    if not token:
        raise StopProcessing("Verification required.", publish_outbound=False)
    async with httpx.AsyncClient(timeout=10) as client:
        response = await client.post(
            os.environ["CHALLENGE_VERIFY_URL"],  # the service's verification endpoint
            data={"secret": os.environ["CHALLENGE_SECRET"], "response": token},
        )
    if not response.json().get("success"):  # field names depend on the service
        raise StopProcessing("Verification failed.", publish_outbound=False)
    return content
```

Webhook runs carry the request in `context["request"]` with lowercase header names; see [verifying the sender](connectors.md#webhook).

## Managing apps

The dashboard's **Apps** page lists each environment's apps with their URL, sign-in, and last-7-day stats; **Details** has **Settings** (custom domain, sign-in), **Stats**, and **Access** (the deployed rules). Viewing needs `apps.view`; changing settings needs `apps.manage`. connic MCP has `list_apps`, `get_app`, `update_app`, and `get_app_stats` (see [platform-mcp.md](platform-mcp.md)). There is no public REST route for apps.

Two per-app settings live only in **Details → Settings** and `update_app`, never in `apps/<key>.yaml`:

- `default_url` (default `true`): an app with a custom domain answers on both its `connic.app` address and the domain. `false` redirects the `connic.app` address to the custom domain, keeping path and query. It needs an active custom domain, and removing the domain turns it back to `true`.
- `powered_by` (default `true`): every HTML page the app serves ends with a small "Powered by connic" line in the page's own font and color. `false` turns it off for that app.
