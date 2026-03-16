# REST API Design — Architecture Notes

---

## Spring MVC Request Processing Pipeline

```
HTTP Request
     │
     ▼
DispatcherServlet
     │
     ├── HandlerMapping.getHandler()
     │   (finds @Controller method matching URL + method)
     │
     ├── HandlerAdapter.handle()
     │   │
     │   ├── @RequestMapping method resolution
     │   ├── Argument resolution (@PathVariable, @RequestBody, @RequestParam)
     │   │   └── HttpMessageConverter reads request body (JSON → Java object)
     │   │
     │   ├── Method invoked
     │   │
     │   └── Return value handling
     │       └── HttpMessageConverter writes response (Java → JSON)
     │
     └── Response written to client
```

---

## Content Negotiation — How Spring Chooses Response Format

```
Client: Accept: application/json
Server logic:
1. Find all HttpMessageConverters that can write the return type
2. Filter by Accept header media type
3. Use first matching converter
4. Default to application/json if no Accept header

Common converters:
- MappingJackson2HttpMessageConverter → JSON (Jackson)
- Jaxb2RootElementHttpMessageConverter → XML
- StringHttpMessageConverter → text/plain
- ByteArrayHttpMessageConverter → application/octet-stream
```

---

## ResponseEntity vs @ResponseBody vs @RestController

```java
// @RestController = @Controller + @ResponseBody on all methods
// Response body is the return value, serialized by HttpMessageConverter

@RestController
public class ProductController {

    // Returns JSON by default (via @RestController)
    @GetMapping("/products/{id}")
    public ProductResponse getProduct(@PathVariable Long id) {
        return new ProductResponse(...);  // Serialized to JSON, 200 OK
    }

    // ResponseEntity gives full control over status, headers, body
    @PostMapping("/products")
    public ResponseEntity<ProductResponse> createProduct(...) {
        return ResponseEntity
            .created(location)         // Status 201
            .header("X-Custom", "val") // Custom header
            .body(new ProductResponse(...));
    }

    // Return void with status
    @DeleteMapping("/products/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();  // 204
    }
}
```

---

## Filter vs Interceptor vs AOP for Cross-Cutting Concerns

```
Request → [Filters] → [DispatcherServlet] → [Interceptors] → [Controller] → [AOP]

Filters (javax.servlet.Filter):
  - Operate at Servlet level
  - See raw HttpServletRequest/Response
  - Can modify request/response bytes
  - Run for ALL requests (even static files)
  - Good for: authentication, CORS, logging, compression

HandlerInterceptors:
  - Operate at Spring MVC level
  - Have access to handler (controller method)
  - preHandle / postHandle / afterCompletion
  - Only run for requests handled by DispatcherServlet
  - Good for: authorization, audit logging, locale handling

AOP (@Around):
  - Operate at method invocation level
  - Full access to method args and return value
  - Can wrap any Spring bean method
  - Good for: caching, transactions, metrics, retry
```
