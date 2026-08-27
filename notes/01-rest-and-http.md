# 01 — REST & HTTP

## REST API

**REST (Representational State Transfer)** is an architectural style for designing APIs around resources, commonly using HTTP.

```text
GET    /users
GET    /users/123
POST   /users
PUT    /users/123
PATCH  /users/123
DELETE /users/123
```

- URL identifies the resource.
- HTTP method describes the intended operation.
- REST is not HTTP: HTTP is the protocol; REST is an architectural style.
- Important characteristics: resource-oriented, stateless, uniform interface, resource representations such as JSON.

## HTTP Methods

| Method | Meaning | Safe? | Idempotent? |
|---|---|---:|---:|
| GET | Retrieve | Yes | Yes |
| POST | Create / trigger operation | No | No (generally) |
| PUT | Replace resource at URI | No | Yes |
| PATCH | Partially modify | No | Depends on operation |
| DELETE | Delete | No | Yes |

### Safe vs Idempotent

**Safe** means the request does not intentionally modify server state.

**Idempotent** means repeating the same request has the same intended effect as making it once. Responses do not have to be identical.

### PUT vs PATCH

```text
POST /users       -> commonly creates; server chooses ID
PUT  /users/{id}  -> full replacement/update
PATCH /users/{id} -> partial update
```

HTTP semantics allow PUT to create a resource when the client specifies the target URI and it does not exist. An application can choose to reject that case.

Common success codes:

- PUT: `200`, `204`, or `201` if it creates.
- PATCH: `200` or `204`.

PATCH idempotency depends on the operation:

```text
PATCH { "name": "Tanay" }          -> can be idempotent
PATCH { "operation": "increment" } -> non-idempotent
```

### POST idempotency

POST is generally non-idempotent because retries can create repeated effects. For operations such as payments, an **idempotency key** can let the server recognize a retry and avoid performing the operation twice.

### DELETE idempotency

DELETE is idempotent even if responses differ:

```text
1st DELETE -> 204
2nd DELETE -> 404
```

The intended final state is the same: the resource does not exist.

## HTTP Status Codes

Status codes describe the **result**, not the HTTP method.

### Common 2xx

- `200 OK` — successful request.
- `201 Created` — resource created.
- `202 Accepted` — accepted for asynchronous processing.
- `204 No Content` — successful request with no response body.

### Common 4xx

- `400 Bad Request` — malformed/invalid request or request validation failure.
- `401 Unauthorized` — authentication missing/invalid.
- `403 Forbidden` — authenticated/understood, but not permitted.
- `404 Not Found` — resource does not exist.
- `409 Conflict` — request conflicts with current resource state, e.g. uniqueness conflict.

## API Versioning

Versioning allows breaking API changes while keeping old clients working.

### URI

```text
/api/v1/users
/api/v2/users
```

### Header

```http
Accept: application/vnd.myapp.v2+json
```

### Query parameter

```text
/api/users?version=2
```

Interview takeaway: version primarily for **breaking changes**. Backward-compatible additions generally do not require a new version.

## Pagination

Pagination prevents APIs from returning unnecessarily large datasets.

### Offset

```http
GET /users?page=2&size=20
```

Conceptually:

```sql
SELECT * FROM users
ORDER BY id
LIMIT 20 OFFSET 20;
```

Pros: simple, supports arbitrary page jumps.

Cons: large offsets can be expensive; results can shift when data changes.

### Cursor / Keyset

```http
GET /users?limit=20&after=<cursor>
```

Conceptually:

```sql
SELECT * FROM users
WHERE id > :lastSeenId
ORDER BY id
LIMIT 20;
```

Pros: efficient for large datasets and more stable with changing data.

Cons: more complex and does not naturally support arbitrary page jumps.

Use deterministic ordering for pagination.

## Common Request Headers

- `Authorization` — credentials, commonly `Bearer <JWT>`.
- `Content-Type` — format of request body, e.g. `application/json`.
- `Accept` — response representation desired, e.g. `application/json`.
- `Cookie` — sends applicable cookies.
- `Origin` — browser request origin; important for CORS.
- `Host` — target host.
- `User-Agent` — identifies client software.

**Interview trap:**

```text
Content-Type -> what I am sending
Accept       -> what I want back
```

## Common Response Headers

- `Content-Type` — response body format.
- `Content-Length` — response body size.
- `Cache-Control` — caching behavior.
- `ETag` — identifier for a particular resource representation/version.
- `Location` — often identifies a newly created resource after `201`.
- `Set-Cookie` — asks browser/client to create or update a cookie.

### ETag / Conditional Requests

Server:

```http
ETag: "v1"
```

Client later:

```http
If-None-Match: "v1"
```

If unchanged:

```http
304 Not Modified
```

The client can use its cached representation instead of receiving the body again.
