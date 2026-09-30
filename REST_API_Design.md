# REST API Design, Validation, Exception Handling, Idempotency

## HTTP methods & idempotency
**Idempotent** = calling it 1 time or 10 times gives the same final state.

| Method | Purpose | Idempotent | Safe (no change) |
|---|---|---|---|
| GET | read | ✔ | ✔ |
| POST | create | ✘ | ✘ |
| PUT | replace whole resource | ✔ | ✘ |
| PATCH | partial update | usually ✘ | ✘ |
| DELETE | remove | ✔ | ✘ |

## Status codes to remember
- **200** OK, **201** Created (+ `Location` header), **204** No Content
- **400** bad input / validation, **401** not logged in, **403** no permission, **404** not found, **409** conflict (duplicate / version), **422** semantic error, **429** too many requests
- **500** server bug, **502/503/504** gateway/service unavailable/timeout

## Layered flow of one request
```
Client
  │  POST /api/orders  (JSON)
  ▼
Filter chain (Security, logging, CORS)
  ▼
DispatcherServlet → HandlerMapping → finds OrderController.create()
  ▼
Message converter (Jackson): JSON → OrderRequest object
  ▼
@Valid  → validation ──fail──► MethodArgumentNotValidException ──┐
  ▼ pass                                                          │
Controller → Service (@Transactional) → Repository → DB           │
  ▼                                                               │
Return object → Jackson → JSON → 201                              │
                                                                  ▼
       any exception ─────────────────────────► @ControllerAdvice handler → clean error JSON
```

## Validation
```java
public record OrderRequest(
    @NotBlank String customerId,
    @Min(1) int quantity,
    @Email String email) {}

@PostMapping
public ResponseEntity<OrderResponse> create(@Valid @RequestBody OrderRequest req) {
    OrderResponse res = service.create(req);
    return ResponseEntity.created(URI.create("/api/orders/" + res.id())).body(res);
}
```

## Global exception handling (must know)
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> notFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> invalid(MethodArgumentNotValidException ex) {
        String msg = ex.getBindingResult().getFieldErrors().stream()
                       .map(e -> e.getField() + ": " + e.getDefaultMessage())
                       .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(new ErrorResponse("VALIDATION_FAILED", msg));
    }

    @ExceptionHandler(Exception.class)          // last resort – never leak stack trace
    public ResponseEntity<ErrorResponse> generic(Exception ex) {
        log.error("Unexpected", ex);
        return ResponseEntity.status(500).body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong"));
    }
}
```
Benefit: controllers stay clean, consistent error format everywhere.

## Idempotent POST (payments!) — very common scenario
Problem: client times out and retries `POST /payments` → double charge.
```
Client sends header  Idempotency-Key: abc-123
   ▼
Server: does key abc-123 exist in DB/Redis?
   ├─ Yes → return the SAME stored response (do not process again)
   └─ No  → process, store (key → response), return
```
Store the key with a unique constraint so two parallel retries cannot both pass.

## More REST topics
- **Versioning:** `/v1/orders` (URI), header, or media type.
- **Pagination:** `?page=0&size=20&sort=createdAt,desc` (`Pageable`). For huge tables use **keyset/cursor** (`WHERE id > lastId LIMIT 20`) – `OFFSET 1000000` is slow.
- **DTO vs Entity:** never expose entities (leaks fields, lazy-loading errors, tight coupling). Use DTOs / MapStruct.
- **Rate limiting:** token bucket (Bucket4j, Redis, API gateway) → 429.
- **`@RestController`** = `@Controller` + `@ResponseBody`.
- **`@PathVariable` vs `@RequestParam`:** `/users/5` vs `/users?id=5`.
- **PUT vs PATCH**, **HATEOAS** (rarely used), **OpenAPI/Swagger** for docs.
