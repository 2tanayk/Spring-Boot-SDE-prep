# Spring Security — Filter Chain, `SecurityContext` & Method Security

## Security Filter Chain

Spring Security processes HTTP requests through a chain of security filters before the request reaches the controller.

Conceptually:

```text
HTTP Request
     ↓
Spring Security Filter Chain
     ↓
Authentication / security processing
     ↓
SecurityContext
     ↓
Controller
```

For a JWT-based REST API, the rough flow is:

```text
Authorization: Bearer <JWT>
          ↓
Security Filter Chain
          ↓
Validate JWT
          ↓
Create Authentication
          ↓
Populate SecurityContext
          ↓
Controller
```

The exact filters depend on the application's Spring Security configuration.

## SecurityContext

`SecurityContext` represents the security information associated with the current execution. Most importantly, it contains the current `Authentication`.

Conceptually:

```text
SecurityContext
      ↓
Authentication
      ├── Principal → current user
      ├── Authorities → roles/permissions
      └── Authenticated → true/false
```

It can be accessed through:

```java
Authentication authentication =
        SecurityContextHolder.getContext().getAuthentication();
```

For example:

```java
String username = authentication.getName();
var authorities = authentication.getAuthorities();
```

### Key relationship

> **Security filters establish the security context; application code consumes it.**

## Authentication vs Authorization

**Authentication:** “Who are you?”

Example: a valid JWT identifies the user as `tanay`.

**Authorization:** “Are you allowed to perform this operation?”

Example: the authenticated user has `ROLE_ADMIN` and can delete users.

The filter chain can participate in authentication and request authorization. Method security can then perform authorization at the method level.

---

# `@PreAuthorize` / Method Security

Enable method security with:

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
}
```

Then protect a method:

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long userId) {
    // ...
}
```

Before the method executes, Spring Security evaluates the expression. If authorization fails, the actual method is not executed.

Conceptually:

```text
Caller
  ↓
Method-security interceptor
  ↓
Evaluate @PreAuthorize
  ↓
Authorized?
  ├── yes → actual method
  └── no  → access denied
```

`@PreAuthorize` relies on the existing `SecurityContext`; it does not authenticate the user itself.

## `hasRole` vs `hasAuthority`

Classic interview distinction:

```java
@PreAuthorize("hasRole('ADMIN')")
```

commonly checks for the authority:

```text
ROLE_ADMIN
```

Whereas:

```java
@PreAuthorize("hasAuthority('ADMIN')")
```

checks for:

```text
ADMIN
```

So, in the common role-prefix setup:

```text
hasRole("ADMIN")
        ↓
ROLE_ADMIN

hasAuthority("ROLE_ADMIN")
        ↓
ROLE_ADMIN
```

But `hasAuthority("ADMIN")` is different.

---

## Request security vs method security

Request-level security protects HTTP endpoints:

```java
.requestMatchers("/admin/**").hasRole("ADMIN")
```

Method security protects the method itself:

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }
```

They can be used together.

```text
HTTP Request
     ↓
Request authorization
     ↓
Controller
     ↓
Service method
     ↓
@PreAuthorize
     ↓
Business operation
```

Method security is especially useful when authorization belongs to the business operation rather than just one HTTP route, because the method may be called from multiple entry points.

## Expression-based / object-level authorization

`@PreAuthorize` can use the current authentication and method arguments.

For example:

```java
@PreAuthorize("#userId == authentication.principal.id")
public void updateUser(Long userId) {
    // ...
}
```

This allows authorization decisions to depend on both the authenticated user and the method input.

---

## Interview mental model

> **Filter Chain = security processing around the HTTP request.**

> **SecurityContext = “who is the current authenticated user?”**

> **Authentication = identity + authorities.**

> **`@PreAuthorize` = authorization check before a method executes.**

> **Request authorization protects endpoints; method security protects operations.**
