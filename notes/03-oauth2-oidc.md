# 03 — OAuth 2.0, OIDC & PKCE

## OAuth 2.0

**OAuth 2.0 is an authorization framework for delegated access.** It allows an application to obtain limited access to protected resources without requiring the user to give the application their password.

### Four actors

1. **Resource Owner** — user who owns the data.
2. **Client** — application requesting access.
3. **Authorization Server** — authenticates/obtains consent and issues tokens.
4. **Resource Server** — API that protects and serves the resource.

```text
Resource Owner
      |
      v
   Client
      |
      v
Authorization Server
      |
  Access Token
      |
      v
Resource Server
```

The Authorization Server issues the token; the Resource Server validates/uses it to authorize access.

## Authorization Code Flow

```text
1. Client redirects user to Authorization Server
2. Authorization Server authenticates user + obtains consent
3. Authorization Server redirects back with authorization code
4. Backend exchanges code at token endpoint
5. Authorization Server returns access token
6. Client uses access token to call Resource Server
```

The authorization code is **not** the access token. It is a short-lived credential exchanged for tokens.

Typical authorization request includes values such as:

```text
client_id
redirect_uri
scope
state
response_type=code
```

## PKCE

**PKCE (Proof Key for Code Exchange)** binds the authorization-code exchange to the client that initiated it.

Client generates a random secret:

```text
code_verifier = random value
```

and derives:

```text
code_challenge = SHA-256(code_verifier)
```

The authorization request contains the challenge. After receiving the authorization code, the client sends the code plus the original verifier to the token endpoint.

The Authorization Server verifies that the verifier matches the original challenge.

```text
code_verifier
      |
  derive/hash
      |
code_challenge ------> Authorization Server

      ... user login ...

authorization_code <--- Authorization Server

code + code_verifier -> Token Endpoint
                         |
                    verify challenge
                         |
                    Access Token
```

If an attacker steals only the authorization code, they do not have the verifier and cannot complete the exchange.

## Scopes

Scopes define the permissions being requested/granted:

```text
scope=profile:read email:read
```

The Resource Server can enforce the required scope for an operation.

## OAuth vs JWT

They are different concepts:

```text
OAuth 2.0 -> authorization framework
JWT       -> token format
```

An OAuth access token **can be a JWT**, but it can also be an opaque token.

## OAuth vs OIDC

OAuth 2.0 is primarily about delegated authorization.

**OpenID Connect (OIDC)** adds an identity/authentication layer on top of OAuth 2.0.

```text
OAuth 2.0 -> delegated authorization
OIDC      -> user identity/authentication on OAuth 2.0
```

OIDC introduces the **ID Token**, which is distinct from an access token.

- **Access Token:** intended to authorize access to a Resource Server.
- **ID Token:** provides identity information to the client/application.

## External OAuth/OIDC + internal JWT

An application can use an external identity provider and then issue its own JWT:

```text
External OAuth/OIDC
       |
External identity established
       |
Your backend
       |
Issue your own JWT
       |
Your APIs
```

This separates external identity from the application's own API authentication/authorization model.

## Interview Quick Recall

> OAuth 2.0 = delegated authorization framework.

> Authorization Code = temporary credential exchanged for tokens; it is not the access token.

> PKCE = proof that the party exchanging the authorization code is the one that initiated the flow.

> OIDC = identity/authentication layer on OAuth 2.0.

> OAuth does not require JWT; an access token can be opaque or JWT-formatted.
