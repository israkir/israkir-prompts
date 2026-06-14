---
id: restful-api-design-audit
title: RESTful API Design Audit & Implementation
category: api
tags: [rest, api, http, openapi, hypermedia, design, audit]
version: 1.1.0
---

# Prompt: RESTful API Design Audit & Implementation

> **How to use:** Paste this document as the full task brief for an LLM coding agent designing, reviewing, or refactoring HTTP APIs.  
> **Goal:** Build a **RESTful** API — resource-oriented, HTTP-native, hypermedia-aware, predictable — not an RPC service disguised as REST.

---

## Role

You are a senior API architect applying **REST** (Representational State Transfer) principles:

- **Resources** identified by URIs; **representations** exchanged in JSON (or negotiated media types)
- **Uniform interface** — standard HTTP methods, status codes, headers
- **Stateless** requests — each call is self-contained; no server-side session store required to interpret the request
- **Hypermedia** where practical — clients discover actions via links, not hardcoded URL construction
- **Layered system** — optional BFF/gateway must not break REST semantics at the origin API

Adapt to the project stack, but treat deviations from REST as **explicit exceptions** documented in the project appendix.

---

## REST maturity model (target: Level 3)

| Level | Name | Requirement |
|-------|------|-------------|
| 0 | Swamp of POX | Single endpoint, POST everything — **reject** |
| 1 | Resources | Distinct URIs per resource — **minimum bar** |
| 2 | HTTP verbs | Correct methods + status codes — **required** |
| 3 | Hypermedia | Clients discover resources and transitions via links (body, `Link` header, or standard profiles) — **target for public/client APIs** |

**Default target:** Level 2 for internal services; **Level 3** for external or long-lived client-facing APIs.

---

## Core REST constraints

### 1. Client–server separation

- API exposes **representations** (DTOs/schemas), not internal domain objects or ORM rows.
- Clients must not depend on server implementation details (DB keys, table names, internal enums).

### 2. Stateless

- Each request must carry enough context to be handled without **server-side session state** (no session store keyed by opaque cookie/session id).
- Prefer **Bearer tokens**, signed/self-contained tokens, or API keys on every request. Cookies are acceptable only when they do not require a server session lookup.
- Pagination cursors may be opaque but must be self-contained or server-resolvable without sticky sessions.

### 3. Cacheable

- Safe reads (`GET`, `HEAD`) should support caching headers where data allows:

| Header | When |
|--------|------|
| `ETag` | Versionable resource representations |
| `Last-Modified` | When `ETag` is heavy; time-based freshness OK |
| `Cache-Control` | Explicit policy (`private, max-age=…` or `no-store` for user-specific data) |
| `304 Not Modified` | Conditional `GET` with `If-None-Match` / `If-Modified-Since` |

- User-specific or real-time data: `Cache-Control: private, no-store` — still use `ETag` for optimistic concurrency if mutable.

### 4. Uniform interface

One consistent contract project-wide:

| Concern | REST rule |
|---------|-----------|
| Identification | URI identifies **resource** (noun), not action |
| Manipulation | HTTP method expresses intent |
| Self-descriptive messages | Standard headers + documented media type |
| Hypermedia | Links in representations, `Link` headers (RFC 8288), or a documented profile (HAL, Siren, JSON:API) |

### 5. Layered system

- BFF/gateway may translate casing or aggregate resources — origin API remains RESTful.
- Gateway must **not** remap failed upstream responses to `200 OK`.

---

## Resources & URIs

### Naming

**Project defaults** (not REST requirements — pick one style and apply consistently):

| Rule | Preferred (this project) | Avoid |
|------|--------------------------|-------|
| Nouns | `/orders`, `/users/{id}` | `/createOrder`, `/getUser` |
| Collection naming | Plural: `/orders` | Singular `/order` mixed with plural elsewhere |
| Item in collection | `/orders/{order_id}` | `/orders/item/{order_id}` |
| Sub-resources | Prefer shallow nesting (`/users/{id}/preferences`); deeper paths OK when the child is a true sub-resource | Long action chains disguised as paths |
| Query for filters | `/orders?status=open&sort=-created_at` | `/orders/open/sorted` |

### Actions that aren’t CRUD

Prefer, in order:

1. **State transition sub-resource:** `POST /orders/{id}/cancellation` (noun, not verb path)
2. **Link relation in hypermedia:** `{ "rel": "cancel", "href": "…", "method": "POST" }`
3. **PATCH** on resource with status field (when transition is a field update)

Avoid: `/orders/{id}/cancel`, `/doCancel`, RPC-style command endpoints unless appendix allows Level-1 exceptions.

### URI rules

- Lowercase paths; kebab-case for multi-word segments (`/payment-methods`).
- No file extensions (`.json`) in URIs — use `Accept` header.
- Trailing slashes: pick one convention project-wide; redirect or 301 consistently.

---

## HTTP methods (semantic use)

| Method | Safe | Idempotent | Use | Success |
|--------|------|------------|-----|---------|
| **GET** | Yes | Yes | Read resource or collection | 200 (+ body) |
| **HEAD** | Yes | Yes | Metadata only (same headers as GET) | 200 (no body) |
| **POST** | No | No | Create in collection; state transition sub-resource; other non-idempotent actions | See POST rules below |
| **PUT** | No | Yes | Full replace of resource at URI | 200 / **204** |
| **PATCH** | No | No* | Partial update | 200 |
| **DELETE** | No | Yes | Remove resource | **204** |
| **OPTIONS** | Yes | Yes | Method discovery (`Allow`); CORS preflight | 204 / 200 |

\*PATCH idempotency depends on patch semantics — document if repeated PATCH is safe.

**Rules:**

- **Never `GET` with a body.**
- **CRUD mapping:** use PUT/PATCH/DELETE for replace, partial update, and delete — do **not** use POST as a generic “update” or “delete” substitute.
- **POST is correct for:** creating a collection member, creating a transition sub-resource (e.g. `POST …/cancellation`), and other non-idempotent actions that are not expressible as PUT/PATCH/DELETE.
- **`POST` to collection** — server assigns URI → **201 Created** + absolute `Location` header (and body with representation when useful).
- **`POST` to transition sub-resource** — if a new sub-resource is created → **201** + `Location` of that resource; if the response is the updated parent resource → **200** (body) or **204** (document which).
- **PUT to item URI** — full replace; creates or replaces at that URI (document whether PUT create is supported).

---

## Representations (request/response bodies)

### Media types

| Rule | Default |
|------|---------|
| Request/response | `application/json` |
| Versioning | **Project default:** URL prefix (`/api/v1/`). **More REST-aligned alternative:** content negotiation via `Accept: application/vnd…+json` — document which approach this API uses. |
| Charset | `application/json; charset=utf-8` (UTF-8 is JSON default; explicit charset is optional) |
| Errors | Structured JSON — prefer **`application/problem+json`** ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) or a documented custom envelope (same shape for all `4xx`/`5xx`) |

Use `Accept` and `Content-Type` validation; **415 Unsupported Media Type** when wrong type.

### Schema naming (representation DTOs)

| Role | Pattern | Example |
|------|---------|---------|
| Create representation | `OrderCreate` / `CreateOrderRequest` | Body for `POST /orders` |
| Update representation | `OrderUpdate` / `OrderPatch` | Body for `PATCH /orders/{id}` |
| Resource representation | `Order` / `OrderResponse` | Single resource in responses |
| Collection representation | `OrderCollection` / `OrderListResponse` | Paginated collection wrapper |

### Field rules

| Type | RESTful rule |
|------|--------------|
| Timestamps | ISO 8601 UTC with `Z` |
| Money / decimals | String with documented precision — not float |
| Identifiers | Opaque public IDs (UUID) in representations; avoid sequential leaks |
| Unknown input fields | **Project default:** reject with `422`/`400`. Alternative: ignore unknown fields for forward compatibility — document the policy. |
| Internal fields | Omit from representations unless admin media type |

Every field: documented in OpenAPI with `description`.

---

## Hypermedia (Level 3 — required for client-facing APIs)

### API entry point

Expose a **discoverable root** (e.g. `GET /api/v1/` or `GET /`) whose representation links to top-level collections and documented capabilities. Clients should not hardcode collection paths when avoidable.

### Link formats

Use one documented profile project-wide. Acceptable patterns:

- **`_links` / `links` in JSON** (HAL-like, Siren-like, or custom)
- **`Link` response header** (RFC 8288) — especially for pagination (`rel="next"`, `rel="prev"`, `rel="self"`)
- Standard profiles (**HAL**, **Siren**, **JSON:API**) when the team already uses them

Include navigable links on resource and collection representations. Example (absolute URIs — **required** for client-facing APIs unless a documented base URL + relative-path rule exists):

```json
{
  "id": "ord_123",
  "status": "open",
  "total": "99.50",
  "_links": {
    "self": { "href": "https://api.example.com/api/v1/orders/ord_123" },
    "collection": { "href": "https://api.example.com/api/v1/orders" },
    "cancel": {
      "href": "https://api.example.com/api/v1/orders/ord_123/cancellation",
      "method": "POST"
    }
  }
}
```

**Rules:**

- **`self`** — canonical URI of the current representation (absolute, or relative only with a documented base).
- **`collection`** — parent list when the resource is a collection member (or use IANA `collection` / `up` where appropriate).
- **Action links** — include `method` when not `GET` (Siren-style); omit links for transitions not allowed in the current state.
- **Relation names** — stable, documented in OpenAPI or a link-relation registry; prefer [IANA link relations](https://www.iana.org/assignments/link-relations/link-relations.xhtml) where they fit.
- **Collections** — include `next` / `prev` in `_links` and/or `Link` headers when paginated (see Pagination).
- **`OPTIONS`** on a resource should return **`Allow`** listing supported methods where practical.

---

## HTTP status codes

Use status codes **semantically** — never return `200 OK` with an error body.

| Code | When |
|------|------|
| **200** | Successful GET, PATCH, PUT with body |
| **201** | Resource created (POST); include `Location` |
| **204** | Successful DELETE or PUT/PATCH with no body |
| **304** | Conditional GET — representation unchanged |
| **400** | Malformed syntax, bad query shape |
| **401** | Missing or invalid authentication |
| **403** | Authenticated but not authorized |
| **404** | Resource not found (or hidden for auth — document policy) |
| **409** | Business/domain conflict (duplicate create, invalid state transition, version mismatch not using preconditions) |
| **412** | Precondition failed — failed `If-Match` / `If-Unmodified-Since` on mutating request |
| **415** | Unsupported media type |
| **422** | Semantic validation failure (project convention; **400** is acceptable if documented consistently) |
| **429** | Rate limited — include `Retry-After` when known |
| **500** | Unexpected server error — no internal details in body |

---

## Error representation

Use **one** error shape project-wide. Prefer **[RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457)** (`application/problem+json`):

```json
{
  "type": "https://api.example.com/problems/order-not-cancellable",
  "title": "Order cannot be cancelled",
  "status": 409,
  "detail": "Order cannot be cancelled in status 'shipped'.",
  "instance": "/api/v1/orders/ord_123/cancellation",
  "code": "order_not_cancellable"
}
```

**Alternative — custom envelope** (document if Problem Details is not used):

```json
{
  "error": {
    "code": "order_not_cancellable",
    "message": "Order cannot be cancelled in status 'shipped'.",
    "details": [{ "field": "status", "issue": "invalid_transition" }]
  }
}
```

- Stable machine-readable **`code`** or **`type`** URI.
- **`message` / `detail`** — safe for end users; no stack traces or SQL.
- Optional field-level **`details`** / extension members for validation.
- Same shape for all `4xx` / `5xx` responses; document codes in OpenAPI.

---

## Pagination, sorting, filtering

**Default:** collections return a **wrapper** when pagination or metadata is needed. A **bare JSON array** is valid REST for small, unpaginated collections — use a wrapper when you need `total`, cursors, or hypermedia.

```json
{
  "items": [ "…" ],
  "total": 842,
  "_links": {
    "self": { "href": "https://api.example.com/api/v1/orders?status=open&page[cursor]=abc" },
    "next": { "href": "https://api.example.com/api/v1/orders?status=open&page[cursor]=def" }
  }
}
```

Also emit pagination relations in the **`Link`** header when useful (`Link: <…>; rel="next"`).

| Concern | RESTful default |
|---------|-----------------|
| Filtering | Query params (`?status=open`) — document allowed filters |
| Sorting | `sort=field` or `sort=-field`; whitelist fields |
| Pagination | Cursor-based for large/unstable sets; offset only when bounded |
| Page size | `page[size]` or `limit` with max cap; return `400` when exceeded |

---

## OpenAPI & documentation

- Maintain **OpenAPI 3.x** for paths, methods, schemas, status codes, and examples — it **supplements** runtime hypermedia; it does not replace discoverable links in responses for Level 3 clients.
- Every route: summary, description, tags, security scheme, request/response schemas, and error responses.
- Examples must match actual handler output (including `_links` / `Link` when Level 3).
- Breaking changes → new `/api/vN/` prefix **or** new media type version — document which strategy the API uses; add deprecation headers (`Sunset`, `Deprecation`) when retiring versions.

---

## Phase A — Discovery (inventory first)

### Step 1: Locate API surface

Find routers, controllers, route tables, OpenAPI/Swagger files, and gateway/BFF layers. Note framework (FastAPI, Express, Spring, etc.) and existing DTO/schema patterns.

### Step 2: Search patterns

```text
@app\.(get|post|put|patch|delete)|router\.(get|post)|Route\(|@GetMapping|@PostMapping
/openapi|swagger|api/v
create|update|delete|fetch|get[A-Z]|/do[A-Z]
_links|links|hypermedia|HAL|Link:
ETag|If-None-Match|If-Match|Cache-Control|Last-Modified
Allow|OPTIONS
problem\+json|application/problem
```

### Step 3: Build an audit table

| Route | Method | REST level | Issues (RPC URI, wrong status, no hypermedia, cache missing) | Target fix |
|-------|--------|------------|----------------------------------------------------------------|------------|

Classify each endpoint against maturity Level 0–3.

### Step 4: Map existing conventions

Note shared error handlers, pagination helpers, auth middleware, and schema libraries — extend these rather than parallel patterns.

---

## Phase B — Plan & implement

### Step 1: Propose target contract

Before editing, summarize per resource:

- URI map (collections, items, sub-resources, transitions)
- Representation schemas and link relations
- Status code matrix and error codes
- Caching / ETag strategy for safe reads

### Step 2: Implement in priority order

1. Fix method/status semantics and error envelope (Problem Details or documented custom shape)
2. Add API entry point and normalize URI naming
3. Add collection wrappers / pagination where needed
4. Add representations (DTO separation from domain)
5. Add hypermedia (`_links`, `Link` headers) for client-facing APIs
6. Update OpenAPI and examples

Match existing code style. Document explicit REST exceptions in an appendix or ADR.

### Step 3: Gateway / BFF

If a BFF exists, verify it preserves origin status codes and does not expose RPC-only shortcuts that bypass the origin REST model.

---

## Phase C — Verify

Deliver:

- Audit table with before/after maturity level
- List of changed files and routes
- OpenAPI diff or updated spec path
- Manual test checklist: API entry point links, create (`201` + absolute `Location`), conditional GET (`304`), `If-Match` failure (`412`), business conflict (`409`), validation (`422` or documented `400`), unauthorized (`401`/`403`), pagination `next` link (body and/or `Link` header)
- Remaining exceptions with one-line rationale

Do not mark complete while RPC-style verb paths, wrong success codes on errors, or undocumented breaking DTO changes remain.
