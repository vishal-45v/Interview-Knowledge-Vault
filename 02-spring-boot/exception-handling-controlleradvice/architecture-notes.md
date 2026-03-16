# Exception Handling — Architecture Notes

---

## Exception Flow Through Spring MVC

```
Controller throws Exception
         │
         ▼
HandlerExceptionResolverComposite
  ├── ExceptionHandlerExceptionResolver
  │     (handles @ExceptionHandler methods)
  │     1. Check @ExceptionHandler in current controller
  │     2. Check @ExceptionHandler in @ControllerAdvice beans
  │
  ├── ResponseStatusExceptionResolver
  │     (handles @ResponseStatus on exception class)
  │
  ├── DefaultHandlerExceptionResolver
  │     (handles standard Spring MVC exceptions)
  │
  └── SimpleMappingExceptionResolver (if configured)
        (maps exceptions to view names)
```

---

## Exception Handler Resolution Order

```
1. Most specific exception type wins
   NoSuchElementException handler wins over RuntimeException handler
   when NoSuchElementException is thrown

2. @ControllerAdvice ordering
   @Order(1) runs before @Order(2)
   Lower number = higher priority

3. Controller-local handlers beat @ControllerAdvice
   @ExceptionHandler in @RestController is checked first

4. Within same @ControllerAdvice, most specific type wins
```

---

## RFC 7807 Problem Detail Structure

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "The request body contains invalid fields",
  "instance": "/orders/create",
  "timestamp": "2024-01-15T10:30:00Z",
  "errors": [
    {"field": "quantity", "message": "must be greater than 0"}
  ]
}

Fields:
  type:     URI identifying the problem type (documentation link)
  title:    Human-readable summary (same for same type)
  status:   HTTP status code
  detail:   Human-readable explanation for THIS occurrence
  instance: URI of the specific occurrence
```
