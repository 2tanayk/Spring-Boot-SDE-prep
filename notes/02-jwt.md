# 02 — JWT

## What is JWT?

A **JWT (JSON Web Token)** is a signed token containing claims about an authenticated identity or other token context.

Typical flow:

```text
Client -> login -> Server
                    |
                    | authenticate
                    v
                 create JWT
                    |
Client <- JWT ------+

Client -> Authorization: Bearer <JWT> -> API
                                      |
                                      v
                               validate JWT
                                      |
                                      v
                               Authentication
                                      |
                                      v
                                Authorization
```

## Structure

```text
HEADER.PAYLOAD.SIGNATURE
```

### Header

Describes the token/signing algorithm:

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
- `jti` — unique JWT ID; useful for revocation/blacklisting.
- `iss` — issuer.
- `aud` — intended audience.

## JWT is not encrypted

The header and payload are **encoded, not encrypted**. Anyone holding the token can decode them.

Therefore, do not put secrets such as passwords into a normal JWT payload.

## Signature

The signature is calculated over the header and payload using a signing key.

```text
signature = Sign(header + "." + payload, signing key)
```

The receiver validates the signature. If an attacker changes:

```text
role=USER -> role=ADMIN
```

the signature no longer matches.

Therefore the signature provides **integrity/authenticity**, not confidentiality.

## Symmetric vs asymmetric signing

### HS256 / symmetric

The same shared secret is used to sign and verify.

### RS256 / asymmetric

Private key signs; public key verifies.

```text
Authorization Server
  private key -> sign

Resource Server
  public key  -> verify
```

Asymmetric signing is useful when many services need to verify tokens but should not possess the private signing key.

## JWT validation

Validating a JWT is more than checking the signature. Depending on the application, validate:

```text
signature
+ expiration
+ issuer
+ audience
+ required claims
```

## JWT in Spring Security

Conceptually:

```text
HTTP request
    |
    v
Security Filter Chain
    |
    v
Extract Bearer token
    |
    v
Validate JWT
    |
    v
Create Authentication
    |
    v
SecurityContext
    |
    v
Authorization checks
    |
    v
Controller
```

JWT establishes/verifies identity and claims; authorization is a separate step.

```java
@PreAuthorize("hasAuthority('ACCOUNT_READ_ANY')")
```

checks whether the authenticated principal has the required authority.

## Statelessness and revocation

JWT enables stateless authentication because the API can validate the token without a server-side session lookup for basic authentication.

However, revocation introduces state. For example:

```text
JWT jti = abc123

Revoked JTIs:
abc123
```

The token may have a valid signature and unexpired `exp`, but the server can reject it because its `jti` is revoked.

Therefore:

> JWT enables stateless authentication, but mechanisms such as JTI blacklists introduce server-side state around token validity.

Short-lived access tokens reduce the impact of token theft; refresh tokens can be used to obtain new access tokens.

## Session vs JWT

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

## Interview Quick Recall

> JWT = signed token containing claims.

> Payload is encoded, not encrypted.

> Signature provides integrity/authenticity, not confidentiality.

> JWT enables stateless authentication, but revocation mechanisms introduce state.

> Authentication from JWT and authorization via roles/authorities are separate steps.
