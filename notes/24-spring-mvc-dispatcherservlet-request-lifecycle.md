# 24 — Spring MVC — DispatcherServlet Request Lifecycle

## 1. Big picture

When an HTTP request reaches a Spring Boot REST API, the simplified flow is:

```text
HTTP Request
     ↓
Servlet Container (Tomcat)
     ↓
DispatcherServlet
     ↓
HandlerMapping
     ↓
HandlerAdapter
     ↓
Controller
     ↓
Service
     ↓
Repository
     ↓
Database
     ↓
Controller returns result
     ↓
HttpMessageConverter
     ↓
HTTP Response
```

The `DispatcherServlet` is Spring MVC's **front controller**. It coordinates request processing rather than containing business logic itself.

## 2. Request reaches the servlet container

Spring Boot applications commonly run with an embedded servlet container such as Tomcat.

```text
Client
  ↓
HTTP GET /accounts/42
  ↓
Tomcat
```

Tomcat receives the HTTP request and passes it into the servlet infrastructure.

## 3. DispatcherServlet

Spring Boot registers `DispatcherServlet` as part of the MVC setup.

```text
Tomcat
   ↓
DispatcherServlet
```

Its job is to coordinate finding the appropriate handler, invoking it, and processing the response.

## 4. HandlerMapping

The DispatcherServlet needs to determine which controller/handler should process the request.

For example:

```java
@RestController
@RequestMapping("/accounts")
public class AccountController {

    @GetMapping("/{id}")
    public AccountDto getAccount(@PathVariable Long id) {
        return accountService.getAccount(id);
    }
}
```

For:

```http
GET /accounts/42
```

Spring uses the request mappings to resolve:

```text
GET /accounts/42
       ↓
HandlerMapping
       ↓
AccountController.getAccount()
```

> **HandlerMapping determines which handler/controller should process the request.**

## 5. Request binding

Once the handler is known, Spring binds HTTP request data to the controller method's parameters.

### `@PathVariable`

```java
@GetMapping("/{id}")
public AccountDto getAccount(@PathVariable Long id) {
}
```

Request:

```http
GET /accounts/42
```

Spring extracts `42` and converts it to `Long`, effectively invoking the method with `42L`.

### `@RequestParam`

```java
@GetMapping
public List<AccountDto> search(@RequestParam String status) {
}
```

Request:

```http
GET /accounts?status=ACTIVE
```

Spring binds `status=ACTIVE` to the `status` parameter.

### `@RequestBody`

```java
@PostMapping
public AccountDto create(@RequestBody CreateAccountRequest request) {
}
```

For JSON such as:

```json
{
  "name": "Tanay",
  "currency": "INR"
}
```

Spring uses an `HttpMessageConverter`, typically Jackson for JSON, to deserialize the request body into `CreateAccountRequest`.

> **Request binding converts HTTP data such as path variables, query parameters, and request bodies into Java method parameters.**

## 6. HandlerAdapter

After `HandlerMapping` resolves the handler, Spring uses a **HandlerAdapter** to actually invoke the resolved handler.

The useful interview-level flow is:

```text
DispatcherServlet
      ↓
HandlerMapping
      ↓
HandlerAdapter
      ↓
Controller method
```

You do not need to memorize the adapter's internal implementation for SDE-2 preparation; know its role in the request lifecycle.

## 7. Controller → Service → Repository

The controller typically delegates to the service layer:

```java
@GetMapping("/{id}")
public AccountDto getAccount(@PathVariable Long id) {
    return accountService.getAccount(id);
}
```

The application-level flow is then commonly:

```text
AccountController
       ↓
AccountService
       ↓
AccountRepository
       ↓
PostgreSQL
```

This delegation is application architecture rather than work performed directly by `DispatcherServlet`.

## 8. Response conversion

Suppose the controller returns:

```java
return new AccountDto(42L, "Tanay", "INR");
```

The HTTP response needs JSON. Spring uses an `HttpMessageConverter`, typically Jackson for JSON, to serialize the Java object:

```text
AccountDto
   ↓
Jackson / HttpMessageConverter
   ↓
JSON
   ↓
HTTP Response
```

The useful symmetry is:

```text
Request:  JSON → Java object
Response: Java object → JSON
```

## 9. Filters

A Servlet `Filter` operates at the servlet-container level and can run before the request reaches `DispatcherServlet`.

```text
HTTP Request
     ↓
   Filter
     ↓
DispatcherServlet
     ↓
Controller
```

Filters are part of the **Servlet specification**, not specifically Spring MVC.

Typical uses include low-level request processing, correlation IDs, headers, and request logging.

Spring Security's authentication/authorization processing primarily operates through its security filter chain; security is covered separately.

## 10. Interceptors

Spring MVC provides `HandlerInterceptor` for processing around handler/controller execution.

Conceptually:

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

Common interceptor callbacks are:

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

### Filter vs Interceptor

```text
Servlet level
─────────────
Filter
   ↓
Spring MVC level
───────────────
DispatcherServlet
   ↓
Interceptor
   ↓
Controller
```

**Filter:** servlet-level processing, can operate before `DispatcherServlet`.

**Interceptor:** Spring MVC-level processing around handler/controller execution and has knowledge of the selected handler.

For authentication, remember that Spring Security primarily uses its **filter chain**, not a generic MVC interceptor.

## 11. `@ControllerAdvice` / `@RestControllerAdvice`

Exceptions can be handled centrally rather than repeating `try/catch` logic in every controller.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(AccountNotFoundException.class)
    public ResponseEntity<?> handle(AccountNotFoundException ex) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(...);
    }
}
```

Conceptually:

```text
Controller
    ↓
exception
    ↓
@RestControllerAdvice
    ↓
@ExceptionHandler
    ↓
HTTP error response
```

This provides centralized API exception handling.

## 12. Complete request lifecycle

For an endpoint such as:

```http
GET /accounts/42
```

with:

```java
@GetMapping("/{id}")
public AccountDto getAccount(@PathVariable Long id) {
    return accountService.getAccount(id);
}
```

think:

```text
GET /accounts/42
       ↓
     Tomcat
       ↓
     Filter(s)
       ↓
DispatcherServlet
       ↓
 HandlerMapping
       ↓
 HandlerAdapter
       ↓
Controller method
       ↓
@PathVariable binding → id = 42L
       ↓
Service → Repository → Database
       ↓
Controller result
       ↓
HttpMessageConverter / Jackson
       ↓
JSON HTTP response
```

## Interview Quick Recall

> **DispatcherServlet** is Spring MVC's front controller and coordinates request processing.

> **HandlerMapping** determines which handler/controller should handle the request.

> **HandlerAdapter** invokes the resolved handler.

> **Request binding** maps HTTP data to Java method parameters: `@PathVariable`, `@RequestParam`, `@RequestBody`, etc.

> **HttpMessageConverter** converts request/response bodies; Jackson is commonly used for JSON.

> **Filter** = Servlet-level; can run before `DispatcherServlet`.

> **Interceptor** = Spring MVC-level; runs around handler/controller execution.

> **`@ControllerAdvice` / `@RestControllerAdvice`** provides centralized exception handling.

> The core flow is:

```text
Filter → DispatcherServlet → HandlerMapping → HandlerAdapter → Controller → Service → Repository → DB → HttpMessageConverter → Response
```
