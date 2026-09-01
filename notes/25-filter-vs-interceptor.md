# 25 — Filter vs Interceptor

## 1. Where they sit

The key distinction is **which layer sees the request**:

```text
HTTP Request
     ↓
   Filter
     ↓
DispatcherServlet
     ↓
 Interceptor
     ↓
 Controller
```

- **Filter** → Servlet/container level
- **Interceptor** → Spring MVC level

## 2. Filter

A Servlet `Filter` runs as part of the servlet filter chain and can execute before the request reaches `DispatcherServlet`.

It receives:

```java
HttpServletRequest
HttpServletResponse
FilterChain
```

Example JWT authentication filter:

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain)
            throws IOException, ServletException {

        String token = extractJwt(request);

        if (token != null) {
            UserDetails user = validate(token);

            SecurityContextHolder
                    .getContext()
                    .setAuthentication(...);
        }

        chain.doFilter(request, response);
    }
}
```

The important call is:

```java
chain.doFilter(request, response);
```

which continues the filter chain and eventually allows the request to reach `DispatcherServlet`.

### Typical Filter use cases

- JWT/security processing
- correlation/request IDs
- low-level request logging
- CORS
- headers
- other HTTP/Servlet-level concerns

Filters are part of the **Servlet specification**, not specifically Spring MVC.

Spring Security primarily operates through its **security filter chain**.

## 3. Interceptor

Spring MVC provides `HandlerInterceptor` for processing around controller/handler execution.

```java
public class AuditInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler) {

        return true;
    }
}
```

An important difference is the `handler` argument. By this point Spring MVC has resolved the handler, so the interceptor can know which controller/method is going to execute.

For example:

```text
GET /users/123
      ↓
HandlerMapping
      ↓
UserController.getUser()
      ↓
Interceptor
```

### Interceptor callbacks

```text
preHandle()
    ↓
Controller
    ↓
postHandle()
    ↓
Response processing
    ↓
afterCompletion()
```

`preHandle()` can prevent controller execution by returning `false`.

### Typical Interceptor use cases

- controller-level audit logging
- request timing/metrics
- user activity tracking
- feature flags
- MVC-specific preprocessing/postprocessing

## 4. Concrete JWT example

Suppose a request arrives:

```http
GET /api/users/123
Authorization: Bearer eyJ...
```

A JWT authentication filter can run first:

```text
HTTP Request
     ↓
JWT Filter
     ↓
extract + validate JWT
     ↓
set Authentication in SecurityContext
     ↓
DispatcherServlet
     ↓
HandlerMapping
     ↓
Controller
```

This is why authentication is naturally implemented in the filter/security-filter layer.

An interceptor runs later, once Spring MVC is processing the request:

```text
HTTP Request
     ↓
Filter(s)
     ↓
DispatcherServlet
     ↓
HandlerMapping
     ↓
Interceptor
     ↓
Controller
```

## 5. The simplest distinction

Think in terms of what each component knows.

### Filter knows

```text
HTTP request
Headers
Cookies
URL
HTTP method
Servlet request/response
```

### Interceptor knows

```text
Everything relevant to the HTTP/MVC request
+
Selected controller/handler
Handler metadata
```

So:

> **Filter = lower-level Servlet/HTTP concern.**

> **Interceptor = Spring MVC/controller concern.**

## 6. Common interview traps

### Which runs first?

```text
Filter
  ↓
DispatcherServlet
  ↓
Interceptor
  ↓
Controller
```

### Which can run before DispatcherServlet?

**Filter.**

### Which knows the selected controller method?

**Interceptor**, because handler resolution has already occurred.

### Can JWT authentication technically be done in an interceptor?

Yes, it is technically possible, but Spring Security uses the **filter chain** because authentication is a cross-cutting HTTP/security concern that should occur before MVC handler execution.

### Is Spring Security an interceptor?

No. Spring Security's main request-processing mechanism is its **filter chain**.

## 7. Interview answer

> A Filter operates at the Servlet layer and can run before the request reaches Spring MVC's `DispatcherServlet`. It is appropriate for HTTP/security concerns such as JWT authentication, CORS, request IDs, and low-level request processing. An Interceptor operates within Spring MVC after handler resolution, so it can access the selected controller/handler method. It is useful for controller-level concerns such as auditing, timing, and MVC-specific preprocessing. Spring Security primarily uses a filter chain because authentication should happen before MVC controller execution.

## Quick Recall

```text
Servlet layer
     ↓
   Filter
     ↓
DispatcherServlet
     ↓
Spring MVC layer
     ↓
 Interceptor
     ↓
 Controller
```

> **Filter = Servlet-level; Interceptor = Spring MVC-level.**

> **Filter can run before DispatcherServlet.**

> **Interceptor runs after handler resolution and can know the selected controller/handler.**

> **Spring Security primarily uses filters, not MVC interceptors.**
