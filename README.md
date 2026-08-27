# Spring Boot SDE-2 Prep

Lean revision notes for SDE-2 internal team-switch interviews, focused on Java + Spring Boot backend development.

## Index

### 1. REST & Backend Concepts
- [01 — REST & HTTP](notes/01-rest-and-http.md)
  - REST fundamentals
  - HTTP methods
  - Safety vs idempotency
  - HTTP status codes
  - API versioning
  - Pagination
  - Common request/response headers
  - ETag / conditional requests

- [02 — JWT](notes/02-jwt.md)
  - JWT structure and claims
  - Signing and verification
  - Symmetric vs asymmetric signing
  - JWT validation
  - Spring Security JWT flow
  - Statelessness and revocation
  - Session vs JWT

- [03 — OAuth 2.0, OIDC & PKCE](notes/03-oauth2-oidc.md)
  - OAuth actors
  - Authorization Code flow
  - PKCE
  - Scopes
  - OAuth vs JWT
  - OAuth vs OIDC
  - External OAuth/OIDC + internal JWT

- [04 — Cookies, CSRF, SOP & CORS](notes/04-cookies-csrf-sop-cors.md)
  - Cookies
  - HttpOnly / Secure / SameSite
  - `credentials: "include"`
  - Same-Origin Policy
  - CORS
  - Credentialed CORS
  - CSRF
  - CSRF vs CORS vs XSS

### 2. DBMS & SQL
- _Not started_

### 3. Spring Boot & Spring Ecosystem
- _Not started_

## Priority

- 🟢 Core — Must know well: mechanics, trade-offs, practical usage
- 🟡 Secondary — Useful after Core is solid
- 🔴 Advanced — Defer unless the interview bar requires it

## Approach

These notes are intentionally concise and interview-oriented. They capture the mechanism, why the concept exists, practical usage, trade-offs, and common interview traps rather than serving as exhaustive documentation.
