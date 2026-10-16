# REST API Design Guidelines

**Status:** Canonical instruction set. Read this before adding, modifying, or removing any HTTP endpoint.
**Trigger:** the global instruction `HTTP handler or OpenAPI changes → read .orly/docs/REST_API_DESIGN_GUIDELINES.md first` fires when the diff touches a handler, `public/openapi.json`, or any `route_*` file. If you're an agent reading this — you got here because that trigger fired. Follow this doc as a checklist, not as background reading.

This is a goal-oriented instruction set. Each rule states the goal it serves so you can apply judgment at the edge cases instead of memorizing exceptions.

---

## Audience — who this binds

This guide binds the author and reviewer in the consuming repository.
Publishing and merge authorization come from that repository's operating model.

- **The agent** runs the Quick checklist below as part of `CHORE(close)` and the §10 pre-PR gate, opens the PR via `gh pr create`, and answers `orly-babysit-prs` review feedback. The agent is the primary enforcer.
- **The human author** follows the same checklist when working without an agent.
- **A reviewer** (human or `/review`) checks the PR against this doc adversarially. A red box on the checklist that's not justified in the PR description is grounds to block merge.

When this doc says "you" it means the agent or the human author — same rules either way. When it says "MUST" it means a missing or wrong implementation blocks merge; "SHOULD" means deviate only with a one-line rationale in the PR description.

---

## Quick checklist — adding an endpoint

Run this checklist at the repository's review boundary before creating a Pull
Request (PR). Every applicable item needs evidence or an owner-approved
exception; an unchecked item cannot silently become a pass.

- [ ] **URL design** — plural noun resource, hierarchical path, no verbs (§1); operation-style `:verb` declared as one of the three allowed categories (§1)
- [ ] **Path params + trailing slash** — `{resource_id}` matches body field name; no trailing slash (§1)
- [ ] **HTTP method** chosen by semantics; PATCH idempotency guarantee stated; `Idempotency-Key` honored if applicable (§2)
- [ ] **Long-running ops** use the canonical `202 + /v1/operations/{id}` shape (§2)
- [ ] **Request body shape** matches the path (no path-param IDs in body) (§3)
- [ ] **Pagination** uses keyset cursors with shared, declared default and maximum bounds (§3)
- [ ] **List envelope** is exactly `{items, total: int|null, next_cursor: string|null}` — no synonyms (§3)
- [ ] **Bulk endpoints** use `207` with the canonical per-item shape (§3)
- [ ] **Null vs omit** — absent optionals omitted, `null` reserved for "explicitly cleared" (§3)
- [ ] **Datetime fields** are int64 epoch ms with `_at` suffix (NOT ISO 8601) (§4)
- [ ] **Duration fields** are integers with `_ms` or `_seconds` suffix; no bare `timeout`/`ttl`/`duration` (§4)
- [ ] **Status codes** — 409 includes `current_state`; 412 includes `etag`; 429 includes `Retry-After` + `X-RateLimit-*` (§4)
- [ ] **ETag/`If-Match`** wired for any resource with realistic concurrent edits (§4)
- [ ] **Error responses** use the registry; `detail` follows hygiene rules (no IDs, no SQL, no paths, ≤200 chars) (§5)
- [ ] **OpenAPI document** matches the repository's canonical source and declared generation or validation checks (§6)
- [ ] **Route registered** in the repository's actual router and discovery graph (§7)
- [ ] **Handler signature** follows its framework and uses the inputs it reads (§8)
- [ ] **Authentication and ownership** are proved on the served route before data access (§7–§8)
- [ ] **Versioning** — added/renamed/removed surface listed in PR description; deprecation uses `Deprecation` + `Sunset` headers; new response fields declare `x-stability` (§9)
- [ ] **Tests** — happy path + one error per refusal + idempotency double-PATCH + `Idempotency-Key` replay (where applicable) + ETag mismatch (§10)
- [ ] **Logging** — sensitive ID values are DEBUG-only or carry `// log-id-allowed:` comment; secret-shaped fields are write-only or one-time-read (§11)
- [ ] **The repository's declared `verify.unit` command is clean** — the route-scope table's exhaustive match and its tests cover the auth gate matrix (§10)
- [ ] **No file over 350 lines** (§10)
- [ ] **`gitleaks detect` clean** (§10)

If you're skimming, the load-bearing sections for handler authoring are §7 (route registration) and §8 (handler signature). Read those carefully every time.

---

## §1 — URL design

### Use plural nouns for resources

```
GET    /products              ← collection
GET    /products/{id}         ← single resource
POST   /products              ← create
PATCH  /products/{id}         ← partial update
DELETE /products/{id}         ← remove
```

NOT `/getProducts`, `/createProduct`, `/product`.

**Goal:** RESTful conventions; OpenAPI generators (Stainless, Speakeasy, OpenAPI Generator) produce clean SDKs from this shape.

### Reflect ownership in the path

```
GET /workspaces/{ws_id}/agents/{agent_id}/events
POST /workspaces/{ws_id}/agents
```

Nested paths express the containment relationship. Don't flatten — `/agents?workspace_id=...` loses the rule that the agent belongs to that workspace.

### No verbs in URLs

- Bad: `/users/{id}/activate`, `/agents/{id}/start`
- Good: `PATCH /users/{id}` body `{status: "active"}`, `POST /agents/{id}/events` (event resource creation)

**Operation-style endpoints** — the `POST /v1/.../approvals/{gate_id}:approve` colon form is allowed ONLY when the action falls into one of these three categories, named explicitly in the OpenAPI `description`:

1. **Idempotent retry** — re-running converges to the same end state (e.g. `:retry`, `:resync`).
2. **Side-effecting RPC** — the action sends/produces something the caller can't undo by editing a state field (e.g. `:send`, `:rotate`, `:revoke`).
3. **Multi-resource transaction** — the action atomically touches resources beyond the path's primary resource.

Anything else MUST be modeled as `PATCH /resource/{id}` with a state field. "Convenience" is not a category. If you can't pick one of the three in one sentence, you don't have an operation.

**Collision check** — before adding `POST /v1/.../{id}:verb`, grep the same resource's schema in `public/openapi.json` for an existing lifecycle field (`status`, `state`, `stage`, `lifecycle_state`). If one exists AND the new `:verb` would set it to a value that field can already hold, the operation is forbidden — use `PATCH /resource/{id}` body `{status: "<verb>"}` instead. Adding `:approve` when `status` already has an `approved` value is the canonical anti-pattern. The PR description MUST state the result of this grep ("no `status` field on `Approval`" or "`Approval.status` exists but `approved` is not a settable value via PATCH because <reason>").

### Path-param naming consistency

`{agent_id}` not `{id}`, not `{agentId}`, not `{aid}`. The path-param name matches the field name in the resource body. One spelling per concept across the whole API.

### Trailing slash

URLs are canonical without a trailing slash: `/v1/agents`, not `/v1/agents/`. Requests with a trailing slash MUST `308 Permanent Redirect` to the canonical form (preserves method + body). No silent 404, no silent rewrite.

### Tag mapping is 1:1 with resource

Every top-level resource maps to exactly one OpenAPI tag, and each tag maps to one `paths/<tag>.yaml` file. Don't split a resource across multiple tags ("Agents" + "AgentEvents"); don't merge resources under one tag ("Workspaces and Agents"). Sub-resources live under the parent tag's file unless that file hits the §6 400-line cap, at which point you split by sub-resource and document the split in `root.yaml`.

### Resource ID lives in path, not body

```http
PATCH /products/123
body: { "name": "new name" }    ← do NOT include "id": 123 in body
```

If the parent ID is in the path, never repeat it in the body.

### Field naming (Microsoft-aligned)

- **Plural for collections, singular for items**
- **`_id` suffix** for identifiers (`product_id`, `agent_id`) — never bare `id` except as the resource's own primary key field
- **`_at` suffix** for datetimes; values are int64 epoch milliseconds (§4)
- **Unit suffix** for durations: `_ms` or `_seconds` (§4) — never bare `timeout`/`ttl`/`interval`/`duration`/`expiration`
- **Adjectives before nouns** (`completed_items`, not `items_completed`)
- **No `is_` prefix** on booleans (`enabled`, not `is_enabled`)
- **Include units** in any quantitative field (`size_in_bytes`, `wall_ms`, `expiration_days`)
- **Banned field names** (not "avoid" — banned): `data`, `payload`, `object`, `response`, `result`, `value`, `info`, `metadata` as standalone field names. They convey nothing and break SDK ergonomics. Use a domain-specific noun.
- **Avoid** brand names and reserved words

---

## §2 — HTTP method semantics

| Method | Use for | Idempotent |
|--------|---------|------------|
| `GET` | Retrieve resource(s) | Yes |
| `POST` | Create new resource (server assigns ID) OR operation-style endpoint | No |
| `PUT` | Replace a resource fully (or upsert when client supplies the ID) | Yes |
| `PATCH` | Partially update a resource | Default: yes. State the guarantee in the spec. |
| `DELETE` | Remove a resource | Yes (idempotent: 204 on already-deleted) |

**PATCH idempotency guarantee** — PATCH MUST be idempotent unless the spec's "Failure Modes" section explicitly declares otherwise with a reason. Idempotent PATCH means: issuing the same body twice in succession produces identical row state and identical 200 responses. A negative test that issues the same PATCH twice and asserts equality is required (§10).

**Idempotency keys for non-idempotent POSTs.** Any POST that creates a billable, externally-visible, or side-effecting resource MUST accept an `Idempotency-Key` request header (UUIDv7 from the client), except the conversation rules below. The server stores `(workspace_id, key) → response` for at least 24 hours and replays the prior response on duplicate. Endpoints exempt: pure RPC reads, internal-only POSTs. List which endpoints honor the key in the OpenAPI description.

**Owner-approved conversation exception.** Fleet and creator-private draft
`/messages` sends require one UUIDv7 `operation_id` in the JSON body. They MUST
reject an `Idempotency-Key` header. Draft creation uses the same body key and
atomically records its optional first message. Authorization and lifecycle
checks apply before replay. The same key, normalized message and actor recover
the accepted message; changed content or actor conflicts. Sends return
`202 {status, sequence, replayed}` for drafts, or the fleet's accepted event
receipt, with `Location` pointing to that conversation's messages collection.
Pending state, answers and usage are read through messages and its authorized
live stream. These resources create neither a public turn-status resource nor
a generic operation. Creator-private text never appears in workspace streams.

**Owner-approved workspace exception.** `POST /v1/workspaces` accepts no replay
key. It requires a tenant-unique name and performs one tenant-scoped insert with
a server-assigned ID. After an uncertain browser response, the dashboard
refreshes every cursor page from `GET /v1/tenants/me/workspaces`; the
command-line interface performs one exact-name GET and does not retry the POST.
A deliberate same-name retry returns 409 and cannot create a second row. Do not
generalize this exception to other POST endpoints.

**When in doubt:** if the client supplies the ID and the body fully describes the resource, use PUT. If the server assigns the ID, use POST. If only some fields change, use PATCH.

### Long-running operations

Endpoints that can't complete inside the request window MUST follow this convention, except conversation acceptance described above:

1. The kicking POST returns `202 Accepted` with `Location: /v1/operations/{operation_id}` and a body containing `operation_id` and `status: "pending"`.
2. `GET /v1/operations/{operation_id}` returns:
   ```json
   { "operation_id": "...", "done": false, "status": "running", "started_at": 1735689600000 }
   ```
   When `done: true`, the body carries either `result` (the final resource, inline) or `error` (an RFC 7807 envelope per §5).
3. Terminal states are `succeeded`, `failed`, `cancelled`. No others.

If you need a different shape, amend this doc first. Do not invent a parallel polling URL.

---

## §3 — Request body shape

### List response envelope

```json
{
  "items": [ {...}, {...} ],
  "total": 42,
  "next_cursor": "eyJ0Ijoi..."
}
```

The exact key set is `items`, `total`, `next_cursor` — no synonyms. `results`, `data`, `entries`, `nodes`, `records` are forbidden.

Security-bound exception: `GET /v1/tenants/me/workspaces` also returns the
authoritative `tenant_id`. Browser and command-line clients persist that value
with the workspace list so a refreshed identity cannot mix local state from
two tenants.

- `items` — always present, always an array. Empty result is `200 {"items": []}` — never `204`.
- `total` — always present as an `integer | null` field. `null` means "not computed" (page-bounded count would have been too expensive). The OpenAPI schema MUST declare `nullable: true`. Removing the field entirely is forbidden — it forces SDK consumers into branches.
- `next_cursor` — opaque string when more pages exist, `null` on the last page. Always present.

### Single resource response

The resource itself, no envelope:

```json
{
  "id": "01HZQ...",
  "name": "platform-ops",
  "created_at": 1735689600000
}
```

NOT `{ "data": { ... } }`. NOT `{ "resource": { ... } }`. The top-level response IS the resource.

### Filtering, sorting, pagination

```
GET /products?status=active&sort=-created_at&starting_after=01HZQ...&limit=50
```

- **Filtering grammar — exactly one form per case:**
  - **Equality:** `?status=active` (single value), `?status=active,paused` (multi-value, comma-separated, single key).
  - **Time ranges:** `?created_after=<ts_ms>&created_before=<ts_ms>`. Bracket grammar (`?created_at[gte]=...`) is forbidden.
  - **No boolean explosions.** Don't add `?include_x=true&include_y=true` — use `?include=x,y` with a documented enum of legal values, OR don't expose a knob.
- **Sorting:** `sort=field` ascending; `sort=-field` descending. Single sort key per request — no multi-key.
- **Pagination — keyset.** Request: `?starting_after=<resource_id>&limit=<int>`;
  response: `next_cursor: <resource_id> | null`. Send `next_cursor` back as the
  next request's `starting_after`; use shared repository parsing and constants
  for the default and maximum, with bounds tested at the served endpoint.
- **Sparse fieldsets / `?include=` / `?fields=`:** not supported in v1. If you need to slim a payload, design a smaller endpoint. Don't invent.

### Bulk operations

If an endpoint accepts a batch (e.g. `POST /agents/batch`), it MUST return `207 Multi-Status` with this exact shape — no per-endpoint reinvention:

```json
{
  "items": [
    { "id": "01HZQ...", "status": "succeeded" },
    { "id": "01HZR...", "status": "failed", "error": { "type": "...", "title": "...", "status": 409, "detail": "..." } }
  ]
}
```

Per-item `status` ∈ `{succeeded, failed, skipped}`. Per-item `error` follows the §5 RFC 7807 envelope (without `request_id` — that's on the outer response). Whole-request 4xx still applies for malformed batches.

### Null vs omitted fields

- **Optional fields the server doesn't have a value for: omit the key.** Don't serialize `null` for "not present."
- **`null` is reserved for "explicitly cleared."** A PATCH with `{ "label": null }` clears the label; a GET response containing `"label": null` means the row has been cleared.

This rule is binding on responses AND requests. SDK consumers branch on `field === undefined` vs `field === null` — getting it wrong breaks them silently.

### IDs

- Preserve the repository's documented public identifier format and test its
  parsing at the endpoint boundary. Do not substitute another product's format.
- Do not expose database serial integers.
- Sensitive IDs (workspace_id, agent_id) live in the path, not the body. Never log their values at INFO level (§11).

---

## §4 — Response shape

### Success — return the wire type

Return the repository's typed response through its framework's response
renderer. Share the public response shape with its schema source so field names,
status and content type stay consistent.


### Status codes

| Code | When |
|------|------|
| `200` | Successful read or update |
| `201` | Resource created |
| `202` | Accepted for async processing |
| `204` | Successful delete or no-content op |
| `400` | Client error in request shape |
| `401` | Missing or invalid authentication |
| `403` | Authenticated but unauthorized |
| `404` | Resource not found |
| `409` | State-transition conflict — the resource's current state forbids the requested transition. Body MUST include `current_state` in `detail` or as an extension field. |
| `412` | Precondition failed — `If-Match` ETag mismatch. Body MUST include the current `etag` so the client can refetch. |
| `413` | Payload too large |
| `429` | Rate limited — MUST include `Retry-After` header (seconds, integer) and `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` (epoch seconds). |
| `500` | Internal server error |

`409` and `412` are distinct: `409` is "the state of the resource forbids this"; `412` is "your version of the resource is stale." Don't merge them.

If you find yourself wanting another status code, check RFC 7231/7807 — don't invent one.

### Datetime fields

**Integer milliseconds since Unix epoch (UTC), serialized as JSON number, OpenAPI `type: integer, format: int64`.** Field name uses the `_at` suffix.

```json
"created_at": 1735689600000
```

NOT ISO 8601 strings, NOT seconds, NOT floats, NOT a string-encoded integer. This matches the codebase (`EXTRACT(EPOCH FROM created_at) * 1000` in handlers; `i64` in response structs; `format: int64` in `components/schemas.yaml`). Anyone proposing ISO 8601 string datetimes for a new endpoint is wrong — match the existing convention or amend this rule first.

When the field has not yet been set (e.g. an unrevoked key's `revoked_at`), serialize as JSON `null` and declare `nullable: true` in the schema.

### Duration / interval / timeout fields

**Integer with explicit unit suffix in the field name.** Allowed suffixes:

- `_ms` — milliseconds (default for sub-second to multi-second internal timeouts).
- `_seconds` — seconds (TTLs, intervals where Redis or another sub-system uses seconds natively).

Banned:

- Bare names: `timeout`, `ttl`, `interval`, `duration`, `expiration`. The unit MUST be in the name.
- ISO 8601 durations (`PT5M`, `PT1H30M`). Forbidden in both requests and responses.
- Floats. Use the smaller unit if you need sub-integer precision.
- Mixing units in a single endpoint (e.g. one field in `_ms`, a peer in `_seconds`) without a stated reason in the OpenAPI description.

Pick `_ms` unless the underlying system speaks seconds (Redis `EX`, JWT `exp`). When in doubt, `_ms`.

### ETags and optimistic concurrency

For any resource where concurrent edits are realistic (anything mutable that two principals can touch), the `GET` and the response of `PATCH`/`PUT` MUST include an `ETag` response header. The client MUST send `If-Match: <etag>` on subsequent `PATCH`/`PUT`/`DELETE`. Mismatch → `412`. Resources without realistic concurrent edits (workspace-private append-only logs, single-tenant config) may opt out — note that decision in the spec's "Failure Modes" section.

---

## §5 — Error handling

### Use the error registry

Use the repository's error registry and renderer for status, title and public
documentation links. Handlers supply the code and a safe human-readable detail;
they do not duplicate the registry's response shape.


Never assemble a `Problem` by hand and never write a status and body directly:
the status, title and documentation link come from the registry entry, and a
hand-built body is how two spellings of one error reach clients.

If you need an error code that doesn't exist:

1. Add it to the registry with status, title, docs_uri.
2. Add a corresponding test asserting the registry entry is reachable.
3. Document in your spec's "Error Contracts" table.

### Registry `title` style

Titles are **short imperative noun phrases, 2–5 words, sentence case, no trailing punctuation**. Match the existing codebase style:

- Good: `Invalid UUID canonical format`, `Database unavailable`, `Insufficient role`, `Agent name already exists`, `Invalid webhook signature`.
- Bad: `error: database is currently unavailable.` (sentence, lowercase, period), `WORKSPACE_NOT_FOUND` (screaming snake, that's the code not the title), `Something went wrong while processing your request` (vague + long).

The title must be safe to render verbatim in a UI toast — think "what would I show a user."

### `detail` field hygiene

`detail` is user-facing. It MUST follow these rules — not "should":

1. **Length:** ≤200 characters. If you need more, you're explaining implementation; cut it.
2. **Voice:** one sentence OR one fragment. Be consistent within a code: pick one shape for `ERR_INVALID_REQUEST` and stick to it.
3. **Audience:** a developer integrating the API. Not a database admin, not a Zig engineer.
4. **Templating:** `{s}` placeholders are allowed for **enumerable safe values** — role labels, sort options, supported services. `{s}` placeholders are forbidden for **entity values** — never interpolate a `workspace_id`, `agent_id`, `user_id`, email, IP, hostname, or any UUID into `detail`.
5. **Ban list — `detail` MUST NOT contain any of these substrings, even when paraphrased:**
   - Internal table/column names: `pg_`, `pg.`, table names from `schema/*.sql`, column names not exposed in the OpenAPI response.
   - SQL fragments: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `WHERE`, `JOIN`, `CONSTRAINT`, `relation does not exist`, `duplicate key value violates`.
   - Code internals: `panic`, `unwrap`, `error.`, a source path, a file extension, a line number, file paths starting with `/`, allocator names, struct names from non-public modules.
   - Stack traces, addresses (`0x...`), thread IDs, request-internal pointer values.
   - Secret-shaped strings: anything matching `sk_`, `pk_`, `Bearer `, `eyJ` (JWT), `op://`, hex blobs >16 chars.
   - Entity values: literal UUIDs, emails, raw bearer tokens, vault key names, file paths from the workspace.
6. **Acceptable shapes — match these patterns:**
   - Validation: `"<field> must <constraint>"` — `key_name must be 1-64 chars, alphanumeric + hyphen + underscore`.
   - Capability: `"<resource/action> <verb-form>"` — `Workspace access denied`, `Tenant context required`.
   - State: `"<noun> already exists"` / `"<noun> not found"` / `"<noun> expired"` — `Agent name already exists`, `token expired`.
   - Format help: `"<param>: use <format>"` — `invalid_since_format: use Go-style duration (15s, 30m, 2h, 7d) or RFC 3339 (YYYY-MM-DDTHH:MM:SSZ)`.

When writing a refusal, read the closest existing handler and the repository's
canonical renderer before choosing the shape.

### Internal 500s — direct calls

A datastore or service failure uses the repository's canonical refusal renderer.
Name the failed operation in internal diagnostics without exposing its details
to the caller.


The operation constant is what identifies the failure in logs and telemetry;
the caller gets the registry's 500 and no internal detail.

### Error response body (RFC 7807)

```json
{
  "type": "https://docs.agentsfleet.net/errors/invalid-request",
  "title": "Invalid request",
  "status": 400,
  "detail": "workspace_id must be a valid UUIDv7",
  "request_id": "req_01HZQ..."
}
```

`request_id` (carried as `x-e2e-request-id` header on the response) is required for traceability.

**Status-specific extensions** — the envelope is open per RFC 7807 §3.2. Two extensions are mandatory at their corresponding statuses:

- `409` MUST include `current_state: <string>` naming the state that forbade the transition (e.g. `"current_state": "approved"` when the caller tried to approve an already-approved gate).
- `412` MUST include `etag: <string>` carrying the resource's current ETag so the client can refetch and retry.


Don't invent other extensions without amending this doc.

---

## §6 — OpenAPI editing

Locate the repository's canonical OpenAPI source. Update that source and run
its declared generation or validation command; do not assume a Rust daemon,
generated artifact or build target from another product.

## §7 — Registering a route

Register the served method and path in the repository's router and discovery
graph. Prove authentication and resource ownership are enforced before access;
a helper that no served route mounts is not evidence of authorization.

## §8 — Handler signature rule

Follow the repository's framework and read its canonical handler first.
Use validated inputs and the shared response and error renderers.
Enforce authentication and ownership through its canonical mechanism, with tests
against the actual served route; do not duplicate mounted checks by habit.

## §9 — Versioning

URI-based: `/v1/...`. All current endpoints sit under `/v1`. Bump to `/v2` only when a breaking change is unavoidable; default to additive evolution within `/v1`.

Header-based versioning (e.g. `X-API-Version: 2026-04-25`) is NOT used in this project. Don't introduce it.

### What is additive vs breaking

| Change | Class | Allowed within `/v1`? |
|---|---|---|
| New endpoint | Additive | Yes |
| New optional request field | Additive | Yes |
| New response field | Additive | Yes — but mark its stability class (below) |
| Tightening validation (narrower regex, lower max length) | Breaking | No |
| Loosening validation (wider regex, higher max length) | Additive | Yes |
| Adding a new enum value to a request param | Additive | Yes |
| Adding a new enum value to a response field | Breaking for typed SDK consumers — see "Enum extension" below |
| Renaming a field (request or response) | Breaking | No |
| Removing a field | Breaking | No — deprecate first |
| Changing a field's type or unit | Breaking | No |
| Making an optional field required | Breaking | No |
| Making a required field optional | Additive | Yes |
| Changing default pagination `limit` | Breaking (silently changes per-page billing/UI) | No |
| Adding a new error code to an existing endpoint | Additive | Yes |
| Changing a 4xx status code for an existing failure mode | Breaking | No |

### No field renames within `/v1`

Once a field name is exposed, it's immortal until `/v2`. To evolve, add the new field, populate both, mark the old one deprecated (see below), and remove only at the next major bump. Same rule for path-param names and query-param names.

### Enum extension policy

Adding a value to a response enum is a **breaking change for typed SDK consumers** (Go, Rust, TypeScript SDKs generate exhaustive switches; a new value triggers compile errors or silent fallthrough). Two options:

1. **Defer to `/v2`** — preferred when the new value changes semantics meaningfully.
2. **Add as a `additional_*` companion field** — leave the typed enum frozen, add `additional_status: string` for the new value. Document in the OpenAPI description.

If you ship a new enum value inside `/v1`, the PR description MUST acknowledge the SDK regeneration impact and name the consumers that will need a release.

### Deprecation

When deprecating an endpoint or field:

1. Set the response header `Deprecation: true` and `Sunset: <RFC 1123 date>` on every response from the deprecated endpoint (or every response that includes the deprecated field).
2. Minimum 90-day clock between announcing deprecation and removing the endpoint/field. 180 days for anything an external SDK consumer touches.
3. Add a `Link: <docs-url>; rel="deprecation"` header pointing to the migration guide.
4. Document the deprecation in the OpenAPI YAML with `deprecated: true` AND a one-line migration hint in the description.

No silent removals. No removals that skip the `Deprecation` header period.

### Response field stability classes

Every response field has one of three classes, declared via OpenAPI extension `x-stability`:

- `stable` — covered by the deprecation policy above. Default.
- `beta` — may change shape or be removed without the 90-day clock; flagged as `beta` in the OpenAPI description; SDK generators may strip or wrap as optional.
- `internal` — present in responses for tooling/debugging; SDK generators MUST strip; no compatibility guarantee. Avoid leaking these — internal fields tend to leak production state.

Every new response field added in a PR MUST declare a class. Default is `stable` if you forget, which is binding — the deprecation clock starts the moment it ships.

### PR-level surface diff

Every PR that changes the HTTP surface MUST open its description with a "Surface diff" section listing added / renamed / removed `(method, path)` pairs and added / renamed / removed schema fields. This is how reviewers and the orly-babysit-prs loop detect breaking changes that the diff itself buries.

---

## §10 — Pre-PR checklist (testing + ship)

Before opening a PR touching any handler:

- [ ] Declared build and unit commands have observed successful results
- [ ] the declared `verify.unit` command passes — the route-scope match and its tests cover the auth gate matrix
- [ ] the repository's integration suite passes end to end against real datastores, where it declares one
- [ ] Applicable platform targets and test graphs match the repository's declared matrix
- [ ] the declared `verify.lint` command passes — every language gate the repository declares
- [ ] Handler file ≤ 350 lines; split if it grows
- [ ] Integration test covers the happy path AND at least one error path per refusal the handler can return
- [ ] PATCH endpoints have an idempotency test: same body issued twice → identical 200 + identical row state. Skip only if the spec's "Failure Modes" explicitly declares non-idempotent PATCH with a reason (§2)
- [ ] POST endpoints accepting `Idempotency-Key` have a replay test: same key + same body → cached response; same key + different body → 4xx (§2)
- [ ] Mutable resources have an ETag/`If-Match` test: stale `If-Match` → 412 with current `etag` returned (§4)
- [ ] OpenAPI updated — endpoint definition, request/response schemas, error responses
- [ ] `gitleaks detect` clean
- [ ] No new file over 350 lines

---

## §11 — Security

- Use HTTPS for all endpoints. The Cloudflare Tunnel layer (see `playbooks/ARCHITECTURE.md`) enforces this for public ingress.
- Bearer tokens for user-facing endpoints. OAuth (Clerk) for the auth-issuance flow.
- Sensitive IDs (workspace_id, agent_id, user_id) live in the path, not the body. **Never log their literal values at INFO or above.** A log call whose format string OR struct argument references one of these IDs MUST be `DEBUG` or below, OR carry a same-line comment `// log-id-allowed: <reason>` explaining why this specific call is safe (e.g. it's a hashed prefix, not the raw value).
- Use **UUIDv7** for all IDs.
- **Secret-shaped response fields are write-only or one-time-read.** Any field whose name contains `token`, `secret`, `key`, `password`, `credential` (and is not just a key-by-name like `key_name`) MUST either be: (a) write-only — never returned by GET, only echoed in the POST that creates it, or (b) one-time-read — returned in the create response and never again. Document which in the OpenAPI description with `x-secret-handling: write-only | one-time-read`. This includes raw API keys, OAuth tokens, HMAC shared secrets, and webhook signing secrets.
- Never log secret values. Substitution happens at the tool bridge inside the executor sandbox; tokens never enter the agent's context, the event log, or any handler-level log line. (See `docs/ARCHITECTURE.md` §10 for the substrate guarantees.)
- Webhook receivers verify HMAC signatures using `std.crypto.utils.timingSafeEql`. Never string-compare HMACs.
- **CORS posture is per-endpoint and explicit.** Endpoints intended for browser callers (the dashboard at `app.agentsfleet.net`) MUST declare their allowed origin in the spec; everything else MUST refuse cross-origin requests at the edge. Do not enable wildcard `Access-Control-Allow-Origin: *`. Adding browser exposure to an existing endpoint is a surface change — call it out in the §9 PR-level surface diff.

---

## §12 — Performance

Use the consuming repository's declared performance budgets and benchmark lane.
Measure applicable latency percentiles, memory and query bounds under the load
profile the endpoint is expected to serve.


If your endpoint can't meet these in a normal load profile, the spec's "Performance Considerations" section MUST contain ALL of:

1. **Endpoint** — exact `(method, path)`.
2. **Measured numbers** — p50, p95, p99 from a representative load run; cite the bench command and config used.
3. **Load profile** — qps, payload size, concurrency, dataset size that produced the numbers.
4. **Reason** — what makes this endpoint structurally slower (e.g. unavoidable cross-region call, large aggregation, third-party API).
5. **Remediation milestone** — the spec ID where the carve-out is planned to be retired (`M{N}_{NNN}`), OR an explicit `accepted-permanent` with a justification reviewed by the user.

A one-line "this endpoint is slow because reasons" is not a carve-out. Without all five, the carve-out doesn't merge.

---

## §13 — Reference specifications

When the project's conventions don't cover a case, defer to (in priority order):

1. **This document** — project-specific overrides everything.
2. [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) — for naming + pagination semantics.
3. [Google API Design Guide](https://cloud.google.com/apis/design) — for resource modeling.
4. [GitHub REST API Docs](https://docs.github.com/en/rest) — for query-param + filtering patterns.
5. [OpenAPI Specification](https://swagger.io/specification/) — schema validity.
6. [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807) — error response envelope.
7. [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) — newer Problem Details (informational).

If a project rule conflicts with an external guide, the project rule wins; document the conflict in this file.

---

## SDK generation

This API is designed for OpenAPI-driven SDK generation. SDK generators tested against the OpenAPI bundle:

- [Stainless](https://stainlessapi.com/)
- [OpenAPI Generator](https://openapi-generator.tech/)
- [Speakeasy](https://speakeasy.com/)
- [APIMatic](https://www.apimatic.io/)

If you add a non-standard pattern (e.g., a polymorphic response shape) without checking SDK generator output, you risk breaking client codegen on the next bundle. When in doubt, mirror an existing endpoint that's known-clean under codegen.
