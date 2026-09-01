# 26 — `@ControllerAdvice`

## Purpose

`@ControllerAdvice` centralizes controller-level concerns, most commonly **exception handling**, so individual controllers don't need repetitive `try/catch` logic.

For REST APIs, `@RestControllerAdvice` is the convenient variant because it combines `@ControllerAdvice` with `@ResponseBody` behavior.

## Example

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<?> handle(UserNotFoundException ex) {
        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body("User not found");
    }
}
```

If a controller throws:

```text
Controller
    ↓
UserNotFoundException
    ↓
@ControllerAdvice
    ↓
@ExceptionHandler
    ↓
HTTP 404 response
```

## `@ExceptionHandler`

`@ExceptionHandler` tells Spring which method should handle a particular exception type.

You can centralize multiple exception mappings:

```java
@ExceptionHandler(UserNotFoundException.class)
ResponseEntity<?> handleNotFound(...) { ... }

@ExceptionHandler(MethodArgumentNotValidException.class)
ResponseEntity<?> handleValidation(...) { ... }
```

This allows the API to return a consistent error format.

## Interview Quick Recall

> **`@ControllerAdvice`** provides centralized controller advice, commonly global exception handling.

> **`@RestControllerAdvice`** is convenient for REST APIs because it combines `@ControllerAdvice` with `@ResponseBody` behavior.

> **`@ExceptionHandler`** maps an exception type to the method that handles it.

> Main benefit: consistent error responses and no duplicated exception-handling code across controllers.
