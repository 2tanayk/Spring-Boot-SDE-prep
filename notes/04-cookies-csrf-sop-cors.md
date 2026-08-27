# 04 — Cookies, CSRF, SOP & CORS

## Cookies

A **cookie** is small data that a server asks a browser/client to store. The browser can automatically attach applicable cookies to later requests for the relevant domain/path.

After login:

```http
Set-Cookie: sessionId=abc123; HttpOnly; Secure
```

Browser stores:

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

For browser `fetch`, this tells the browser to include applicable credentials, especially cookies, on a cross-origin request:

```javascript
fetch("https://api.example.com/account", {
    credentials: "include"
});
```

It **does not bypass cookie security rules**. `SameSite`, domain/path, `Secure`, and other cookie rules still apply.

For a credentialed cross-origin request, the server also needs appropriate CORS response headers, including:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

`Access-Control-Allow-Origin: *` cannot be used as the wildcard origin for a credentialed CORS request.

Mental model:

```text
credentials: "include"
    -> request-level instruction to include applicable credentials

SameSite / Secure / Domain / Path
    -> cookie-level rules still decide whether a cookie is sent
```

---

## Same-Origin Policy (SOP)

SOP is a browser security policy that generally prevents JavaScript from one **origin** from freely reading data from another origin.

An origin is:

```text
scheme + host + port
```

For example:

```text
https://app.example.com
https://api.example.com
```

are different origins because their hosts differ.

SOP does **not** mean browsers can never send cross-origin requests. The important restriction is that JavaScript is not automatically allowed to read cross-origin responses.

---

## CORS

**CORS (Cross-Origin Resource Sharing)** is a browser-enforced mechanism that lets a server explicitly allow selected origins to access its cross-origin responses.

Request:

```http
Origin: https://app.example.com
```

Response:

```http
Access-Control-Allow-Origin: https://app.example.com
```

If the origin is allowed, the browser permits the requesting JavaScript to read the response.

CORS is primarily a browser concern. Backend-to-backend calls and tools such as Postman do not enforce browser SOP/CORS in the same way.

### Credentialed CORS

For a cross-origin request that needs cookies:

Frontend:

```javascript
fetch("https://api.example.com/account", {
    credentials: "include"
});
```

Backend:

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

`credentials: "include"` does **not** override `SameSite` cookie rules.

---

## CSRF

**CSRF (Cross-Site Request Forgery)** exploits the browser's automatic inclusion of authentication credentials, especially cookies.

### Banking example

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

### CSRF vs CORS

They solve different problems:

```text
CORS
-> controls whether cross-origin JavaScript can READ responses

CSRF
-> protects against forged authenticated state-changing requests
```

CORS is **not** a replacement for CSRF protection. A cross-origin request can potentially reach the server even when the attacker's JavaScript is prevented from reading the response.

### CSRF vs XSS

```text
CSRF -> attacker tricks the browser into making an authenticated request.
XSS  -> attacker gets malicious JavaScript to execute in the trusted site's context.
```

`HttpOnly` helps prevent JavaScript from reading a cookie, while CSRF defenses address forged authenticated requests.

---

## Interview Quick Recall

> Cookie = browser-managed data that can be automatically attached to matching requests.

> `HttpOnly` blocks JavaScript access; `Secure` restricts sending to HTTPS; `SameSite` controls cross-site sending.

> `credentials: "include"` asks the browser to include applicable credentials on a cross-origin fetch; it does not bypass `SameSite`.

> SOP restricts cross-origin JavaScript access.

> CORS selectively allows trusted origins to read cross-origin responses.

> CSRF tricks an authenticated browser into performing an unwanted action, commonly by exploiting automatically attached cookies.

> CORS and CSRF solve different problems; CORS is not a CSRF defense.
