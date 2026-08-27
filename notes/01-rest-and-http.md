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

## 8. Cookies & Browser Authentication

A **cookie** is small data that a server asks a browser/client to store. The browser can automatically attach applicable cookies to later requests for that domain/path.

Example after login:

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure
```

The browser stores:

```text
bank.com -> sessionId=abc123
```

Later:

```http
GET /account
Cookie: sessionId=abc123
```

The server can map the session ID to the authenticated user.

### Important cookie attributes

**`HttpOnly`**

Prevents JavaScript from reading the cookie through APIs such as `document.cookie`. It does **not** prevent the browser from sending the cookie.

**`Secure`**

Cookie is sent only over HTTPS.

**`SameSite`**

Controls when the browser sends the cookie in cross-site contexts.

- `Strict` — strongest cross-site restriction.
- `Lax` — more permissive; common practical default.
- `None` — permits cross-site use; requires `Secure`.

### `credentials: "include"`

For a browser `fetch`, this tells the browser to include applicable credentials, especially cookies, on a cross-origin request:

```javascript
fetch("https://api.example.com/account", {
    credentials: "include"
});
```

It **does not bypass cookie security rules**. `SameSite`, domain/path, Secure, and other cookie rules still apply.

For a credentialed cross-origin request, the server also needs an appropriate CORS response, including:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

`Access-Control-Allow-Origin: *` cannot be used as the wildcard origin for a credentialed CORS request.

---

## 9. SOP & CORS

### Same-Origin Policy (SOP)

A browser security policy that generally prevents JavaScript from one **origin** from freely reading data from another origin.

An **origin** is:

```text
scheme + host + port
```

For example:

```text
https://app.example.com
https://api.example.com
```

are different origins because their hosts differ.

SOP does **not** mean the browser can never send cross-origin requests. The important restriction is that JavaScript is not automatically allowed to read cross-origin responses.

### CORS

**Cross-Origin Resource Sharing** is a browser-enforced mechanism that lets a server explicitly allow selected origins to access its cross-origin responses.

Request:

```http
Origin: https://app.example.com
```

Response:

```http
Access-Control-Allow-Origin: https://app.example.com
```

CORS is primarily a browser concern. Backend-to-backend calls and tools such as Postman do not enforce browser SOP/CORS in the same way.

### CORS vs CSRF

They solve different problems:

```text
CORS
-> Controls whether cross-origin JavaScript can read responses.

CSRF
-> Protects against an attacker tricking an authenticated browser
   into performing an unwanted state-changing request.
```

CORS is **not** a replacement for CSRF protection. A cross-origin request can potentially reach the server even when the attacker's JavaScript is prevented from reading the response.

---

## 10. CSRF

**CSRF (Cross-Site Request Forgery)** exploits the browser's automatic inclusion of authentication credentials, especially cookies.

### Example

User is logged into `bank.com`:

```text
Browser cookie:
sessionId=ABC123

Bank:
ABC123 -> Tanay
```

User visits `evil.com`. The malicious site causes the browser to send:

```http
POST https://bank.com/transfer
Cookie: sessionId=ABC123

amount=10000
toAccount=ATTACKER
```

The attacker does **not** know `ABC123`. The browser attached it automatically because the request targets `bank.com`.

The bank sees an authenticated request and may perform the action.

### Why cookies make this possible

The browser automatically sends applicable cookies. A Bearer JWT in:

```http
Authorization: Bearer <JWT>
```

is normally not automatically attached to arbitrary cross-site requests; application code must supply it.

However, if the JWT is stored in a cookie, the CSRF concern returns because the browser can automatically send that cookie.

### CSRF defenses

- **CSRF token:** legitimate client sends an additional unpredictable token that the attacker cannot obtain.
- **SameSite cookies:** restrict cookie sending in cross-site contexts.
- Other appropriate browser/server security controls.

For a stateless API using a Bearer token in the `Authorization` header, CSRF protection is often disabled because the authentication credential is not automatically attached by the browser in the same way as a session cookie.

### CSRF vs XSS

```text
CSRF -> attacker tricks the browser into making an authenticated request.
XSS  -> attacker gets malicious JavaScript to execute in the trusted site's context.
```

`HttpOnly` helps prevent JavaScript from reading a cookie, while CSRF defenses address forged authenticated requests.

---

## 11. JWT

A **JWT (JSON Web Token)** is a signed token containing claims about an authenticated identity or other token context.

Typical structure:

```text
HEADER.PAYLOAD.SIGNATURE
```

### Header

Describes the token type/signing algorithm:

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

### Payload

Contains claims:

```json
{
  "sub": "123",
  "role": "ADMIN",
  "iat": 1750000000,
  "exp": 1750003600,
  "jti": "abc-123"
}
```

Important claims:

- `sub` — subject/identity represented by the token.
- `iat` — issued-at time.
- `exp` — expiration time.
- `jti` — unique JWT identifier; useful for revocation/blacklisting.
- `iss` — issuer.
- `aud` — intended audience.

### JWT is not encrypted by default

The header and payload are encoded, not encrypted. Anyone who has the token can decode them.

Therefore:

> Do not put secrets such as passwords into a normal JWT payload.

### Signature

The signature is calculated over the header and payload using a signing key.

Conceptually:

```text
signature = Sign(header + "." + payload, signing key)
```

The receiver validates the signature. If someone changes a claim such as:

```text
role=USER
```

to:

```text
role=ADMIN
```

the signature no longer matches.

Therefore the signature provides **integrity/authenticity**, not confidentiality.

### Symmetric vs asymmetric signing

**HS256 / symmetric:** the same shared secret is used to sign and verify.

**RS256 / asymmetric:** private key signs; public key verifies.

With asymmetric signing, resource servers can verify tokens without possessing the private signing key.

```text
Authorization Server
  private key -> sign

Resource Server
  public key  -> verify
```

### JWT validation

Validating a JWT is more than checking the signature. Depending on the application, validate:

```text
signature
+ expiration
+ issuer
+ audience
+ required claims
```

### JWT in Spring Security

Conceptually:

```text
HTTP request
    ↓
Security Filter Chain
    ↓
Extract Bearer token
    ↓
Validate JWT
    ↓
Create Authentication
    ↓
SecurityContext
    ↓
Authorization checks
    ↓
Controller
```

JWT establishes/verifies identity and claims; authorization is a separate step.

For example:

```java
@PreAuthorize("hasAuthority('ACCOUNT_READ_ANY')")
```

checks whether the authenticated principal has the required authority.

### Statelessness and revocation

JWT enables stateless authentication because the API can validate the token without a server-side session lookup for basic authentication.

However, revocation changes this picture.

Example:

```text
JWT jti = abc123

Revoked JTIs:
abc123
```

The token may still have a valid signature and unexpired `exp`, but the server can reject it because its `jti` is revoked.

Therefore:

> JWT enables stateless authentication, but mechanisms such as JTI blacklists introduce server-side state around token validity.

Short-lived access tokens reduce the impact of token theft; refresh tokens can be used to obtain new access tokens.

### Session vs JWT

**Session:**

```text
Client -> session ID -> Server -> session state
```

Pros: easy revocation; server controls authentication state.

Cons: distributed applications need shared session state or another scaling strategy.

**JWT:**

```text
Client -> signed token -> API
```

Pros: easy horizontal scaling; no session lookup required for basic validation.

Cons: revocation is harder; stolen tokens are serious; tokens can become stale and add request size.

---

## 12. OAuth 2.0

**OAuth 2.0 is an authorization framework for delegated access.** It allows an application to obtain limited access to protected resources without requiring the user to give the application their password.

### Four actors

1. **Resource Owner** — user who owns the data.
2. **Client** — application requesting access.
3. **Authorization Server** — authenticates/obtains consent and issues tokens.
4. **Resource Server** — API that protects and serves the resource.

```text
Resource Owner
      ↓
   Client
      ↓
Authorization Server
      ↓
  Access Token
      ↓
Resource Server
```

The Authorization Server issues the token; the Resource Server validates/uses it to authorize access.

### Authorization Code Flow

Typical flow:

```text
1. Client redirects user to Authorization Server
2. Authorization Server authenticates user + obtains consent
3. Authorization Server redirects back with authorization code
4. Backend exchanges code at token endpoint
5. Authorization Server returns access token
6. Client uses access token to call Resource Server
```

The authorization code is **not** the access token. It is a short-lived credential exchanged for tokens.

### PKCE

**PKCE (Proof Key for Code Exchange)** binds the authorization-code exchange to the client that initiated it.

Client generates a secret:

```text
code_verifier = random value
```

and sends a derived value:

```text
code_challenge = SHA-256(code_verifier)
```

with the authorization request.

After receiving the authorization code, the client sends:

```text
authorization_code + code_verifier
```

to the token endpoint. The Authorization Server verifies that the verifier matches the original challenge.

If an attacker steals only the authorization code, they do not have the verifier and cannot complete the exchange.

```text
code_verifier
      ↓
  derive/hash
      ↓
code_challenge ──→ Authorization Server

      ...user login...

authorization_code ←─ Authorization Server

code + code_verifier ─→ Token Endpoint
                         ↓
                    verify challenge
                         ↓
                    Access Token
```

### Scopes

Scopes define the permissions being requested/granted:

```text
scope=profile:read email:read
```

The Resource Server can enforce the required scope for an operation.

### OAuth vs JWT

They are different concepts:

```text
OAuth 2.0 -> authorization framework
JWT       -> token format
```

An OAuth access token **can be a JWT**, but it can also be an opaque token.

### OAuth vs OIDC

OAuth 2.0 is primarily about delegated authorization.

**OpenID Connect (OIDC)** adds an identity/authentication layer on top of OAuth 2.0.

```text
OAuth 2.0 -> delegated authorization
OIDC      -> user identity/authentication on OAuth 2.0
```

OIDC introduces the **ID Token**, which is distinct from an access token.

- **Access Token:** intended to authorize access to a Resource Server.
- **ID Token:** provides identity information to the client/application.

### Common architecture: external OAuth/OIDC + internal JWT

An application can use an external identity provider:

```text
External OAuth/OIDC
       ↓
External identity established
       ↓
Your backend
       ↓
Issue your own JWT
       ↓
Your APIs
```

This separates external identity from the application's own API authentication/authorization model.

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

> Server-provided resource version/state identifier used with `If-None-Match` for conditional requests; unchanged resource can produce `304 Not Modified`.

### Cookies

> Browser-managed data that can be automatically attached to matching requests. `HttpOnly` blocks JavaScript access, `Secure` restricts sending to HTTPS, and `SameSite` controls cross-site sending.

### SOP / CORS

> SOP restricts cross-origin JavaScript access. CORS is the mechanism for selectively allowing origins to read cross-origin responses.

### CSRF

> Attacker tricks an authenticated browser into making an unwanted request, commonly exploiting automatically attached cookies. CSRF token and SameSite are common defenses.

### `credentials: "include"`

> Tells the browser to include applicable credentials on a cross-origin fetch; it does not bypass `SameSite` or other cookie rules.

### JWT

> Signed token containing claims. Payload is encoded, not encrypted. Signature provides integrity/authenticity. JWT can enable stateless authentication, while revocation mechanisms introduce state.

### OAuth 2.0

> Authorization framework for delegated access. Authorization Code + PKCE is the key flow to understand for modern applications.

### OAuth vs OIDC vs JWT

> OAuth 2.0 = authorization framework; OIDC = identity layer on OAuth 2.0; JWT = token format.
