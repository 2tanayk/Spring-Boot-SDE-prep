# 01 — REST & HTTP

## 1. REST API

**REST (Representational State Transfer)** is an architectural style for designing networked APIs around **resources**, commonly using HTTP.

### Core idea

- URL identifies the **resource**.
- HTTP method describes the intended **operation**.

```text
GET    /users
GET    /users/123
POST   /users
PUT    /users/123
PATCH  /users/123
DELETE /users/123
```

REST is not the same thing as HTTP:

- **HTTP** = communication protocol.
- **REST** = architectural style that commonly uses HTTP.

### Important REST characteristics

- **Resource-oriented:** APIs are organized around resources rather than action-style endpoints.
- **Stateless:** each request contains the information required to process it; the server should not depend on client state stored in a particular application instance.
- **Uniform interface:** consistent HTTP/resource semantics.
- **Representation:** resources are transferred through representations such as JSON rather than exposing the server-side object itself.

Statelessness makes horizontal scaling easier because any service instance can handle a request.

---

## 2. HTTP Methods

| Method | Typical meaning | Safe? | Idempotent? |
|---|---|---:|---:|
| `GET` | Retrieve resource | Yes | Yes |
| `POST` | Create / trigger operation | No | No (generally) |
| `PUT` | Replace resource at a specific URI | No | Yes |
| `PATCH` | Partially modify resource | No | Depends on operation |
| `DELETE` | Delete resource | No | Yes |

### Safe vs Idempotent

**Safe:** the request does not intentionally modify server state.

Safe methods include `GET`, `HEAD`, and `OPTIONS`.

**Idempotent:** making the same request multiple times has the same **intended effect** as making it once.

Idempotency does **not** require identical responses. For example, the first request may return `201` while a later identical request returns `200`; the intended final effect can still be idempotent.

### PUT

Typical application API design:

```text
POST /users       -> create; server typically chooses the ID
PUT  /users/{id}  -> full replacement/update
PATCH /users/{id} -> partial update
```

HTTP semantics allow `PUT` to create a resource when the client specifies the target URI and that resource does not exist. However, an application API can choose to reject that case, e.g. with `404`.

Common successful responses:

- `200 OK` — updated resource returned.
- `204 No Content` — update succeeded, no response body.
- `201 Created` — if the PUT operation actually creates the resource.

### PATCH

`PATCH` modifies part of a resource.

Its idempotency depends on the operation:

```text
PATCH { "name": "Tanay" }       -> can be idempotent
PATCH { "operation": "increment" } -> non-idempotent
```

Common successful responses:

- `200 OK` — updated representation returned.
- `204 No Content` — successful update with no body.

### POST and idempotency

POST is generally non-idempotent because repeating it can create repeated side effects:

```text
POST /orders -> Order #101
POST /orders -> Order #102
```

For operations such as payments, retries can cause duplicate effects. An **idempotency key** can allow the server to recognize a retry and return the previously determined result instead of performing the operation again.

### DELETE and idempotency

Deleting an existing resource and then deleting it again can still be idempotent because the intended final state is the same: the resource does not exist.

The responses do not have to be identical:

```text
1st DELETE -> 204 No Content
2nd DELETE -> 404 Not Found
```

---

## 3. HTTP Status Codes

Status codes describe the **result of the HTTP request**. The HTTP method does not dictate one mandatory status code.

### Common 2xx codes

- `200 OK` — request succeeded; usually a response body is returned.
- `201 Created` — a new resource was created.
- `202 Accepted` — request accepted for processing, commonly for asynchronous work.
- `204 No Content` — request succeeded and there is no response body.

### Common 4xx codes

- `400 Bad Request` — request is invalid/malformed or fails request validation at the API boundary.
- `401 Unauthorized` — authentication is required or credentials are invalid/missing.
- `403 Forbidden` — request is understood, but the caller is not permitted to perform it.
- `404 Not Found` — requested resource does not exist.
- `409 Conflict` — request conflicts with the current state of the resource, e.g. a business/resource uniqueness conflict.

### Common method/status conventions

```text
GET
  -> 200 OK

POST
  -> 201 Created
  -> 202 Accepted (async processing)

PUT
  -> 200 OK / 204 No Content (update)
  -> 201 Created (if it creates)

PATCH
  -> 200 OK / 204 No Content

DELETE
  -> 204 No Content (successful deletion)
```

These are conventions based on the outcome, not rigid method-to-status mappings.

---

## 4. API Versioning

Versioning allows breaking API changes while existing clients continue using the older contract.

### URI versioning

```text
/api/v1/users
/api/v2/users
```

Simple and explicit; common in backend APIs.

### Header versioning

```http
Accept: application/vnd.myapp.v2+json
```

Version is represented in the request headers rather than the URI.

### Query parameter versioning

```text
/api/users?version=2
```

Simple, but less commonly preferred than an explicit URI approach.

### Interview takeaway

Version primarily when introducing **breaking changes**. Backward-compatible additions, such as adding an optional response field, generally do not require a new API version.

---

## 5. Pagination

Pagination prevents APIs from returning unnecessarily large datasets.

### Offset pagination

```http
GET /api/users?page=2&size=20
```

Conceptually:

```sql
SELECT *
FROM users
LIMIT 20 OFFSET 20;
```

**Pros**
- Simple.
- Easy to understand.
- Supports jumping to arbitrary pages.

**Cons**
- Large offsets can become expensive because the database may need to walk past many rows.
- Results can shift when data changes between requests.

### Cursor / keyset pagination

Instead of requesting page N, the client supplies a position/cursor representing where to continue.

Example:

```http
GET /users?limit=20&after=<cursor>
```

Conceptually:

```sql
SELECT *
FROM users
WHERE id > :lastSeenId
ORDER BY id
LIMIT 20;
```

**Pros**
- Efficient for large datasets when the cursor/order column is indexed.
- More stable for continuously changing datasets.

**Cons**
- More complex.
- Cannot naturally jump directly to page 500.

### Pagination ordering

Pagination should use deterministic ordering.

Avoid:

```sql
SELECT * FROM users LIMIT 20 OFFSET 20;
```

Prefer:

```sql
SELECT *
FROM users
ORDER BY id
LIMIT 20 OFFSET 20;
```

For keyset pagination:

```sql
SELECT *
FROM users
WHERE id > :lastSeenId
ORDER BY id
LIMIT 20;
```

---

## 6. Common HTTP Request Headers

Headers carry metadata about an HTTP request/response.

### `Authorization`

Carries authentication credentials, commonly a JWT:

```http
Authorization: Bearer <JWT>
```

### `Content-Type`

Describes the format of the **request body**:

```http
Content-Type: application/json
```

### `Accept`

Specifies the response representation the client wants:

```http
Accept: application/json
```

**Interview trap:**

```text
Content-Type -> format of the body being sent
Accept       -> format of the response desired
```

### `Cookie`

Sends cookies from the client/browser:

```http
Cookie: sessionId=abc123
```

Relevant to session-based authentication.

### `Origin`

Identifies the origin from which a browser request originated. Important for CORS and browser security mechanisms.

### `Host`

Identifies the target host, e.g. `api.example.com`.

### `User-Agent`

Identifies the client software making the request.

---

## 7. Common HTTP Response Headers

### `Content-Type`

Describes the format of the response body:

```http
Content-Type: application/json
```

### `Content-Length`

Size of the response body in bytes.

### `Cache-Control`

Controls caching behavior, e.g.:

```http
Cache-Control: max-age=3600
```

### `ETag`

An identifier representing a particular version/state of a resource.

Example:

```http
ETag: "v1"
```

The client can later send:

```http
If-None-Match: "v1"
```

Meaning:

> "I already have version `v1`; tell me if it is still current."

If the resource has not changed, the server returns:

```http
304 Not Modified
```

and does not retransmit the resource body. The client can use its cached copy.

If the resource changed, the server returns the new representation and a new ETag.

```text
ETag
    Server -> Client

If-None-Match
    Client -> Server
```

### `Location`

Often used with `201 Created` to identify the newly created resource:

```http
HTTP/1.1 201 Created
Location: /users/123
```

### `Set-Cookie`

Instructs the browser/client to create or update a cookie:

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure
```

---

## Interview Quick Recall

### REST

> Architectural style for resource-oriented, stateless APIs using a uniform interface, commonly HTTP.

### Safe vs Idempotent

> Safe = does not intentionally modify state. Idempotent = repeated identical requests have the same intended effect as one request.

### PUT vs PATCH

> PUT = full replacement at a specific URI; PATCH = partial modification. PUT is idempotent; PATCH depends on the operation.

### Content-Type vs Accept

> Content-Type = what I am sending. Accept = what I want back.

### ETag

> Server-provided resource version identifier used with `If-None-Match` for conditional requests; unchanged resource can produce `304 Not Modified`.

### Pagination

> Offset is simpler but can degrade with large offsets; cursor/keyset is more scalable for large, changing datasets but is more complex and doesn't support arbitrary page jumps naturally.
