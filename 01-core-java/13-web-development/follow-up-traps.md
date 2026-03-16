# Web Development — Follow-Up Traps

## Trap 1: Is a Servlet Thread-Safe?

**Trap**: "Servlets are objects, so they must be thread-safe."

**Reality**: Servlets are NOT thread-safe by default, and this is a critical production hazard. The Servlet container creates a single instance of each Servlet and routes all concurrent requests to that one instance by calling `service()` from multiple threads simultaneously.

Any instance variable in a Servlet is shared across all concurrent requests:

```java
// DANGEROUS — threadCount is shared state, race condition
public class CounterServlet extends HttpServlet {
    private int requestCount = 0;  // shared by ALL threads!

    protected void doGet(HttpServletRequest req, HttpServletResponse res) {
        requestCount++;  // NOT atomic — multiple threads corrupt this
        res.getWriter().write("Requests: " + requestCount);
    }
}

// SAFE — use local variables (stack-allocated, per-thread)
public class SafeServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse res) {
        String userId = req.getParameter("userId");  // local — safe
        User user = userService.findById(Long.parseLong(userId));  // local — safe
        res.getWriter().write(user.getName());
    }
}
```

The rule: keep Servlets stateless. No instance fields that are request-specific. Use `AtomicInteger` for shared counters if needed. Spring's `@RestController` beans are also singletons by default — the same thread-safety rule applies.

---

## Trap 2: Filter vs Interceptor — Which Runs First?

**Trap**: "They run at the same time — both intercept requests."

**Reality**: Filters ALWAYS run before Interceptors. The order is:

`Request → Servlet Filter(s) → DispatcherServlet → Spring HandlerInterceptor(s) → @Controller`

Filters are part of the Servlet specification (`javax.servlet.Filter`) and are managed by the Servlet container (Tomcat). They run before Spring MVC even sees the request.

Interceptors are a Spring MVC concept (`HandlerInterceptor`) — they run inside the DispatcherServlet after Spring has matched the request to a handler.

Practical consequences:
- Filters can modify the raw request/response bytes (useful for GZIP compression, request body logging).
- Filters run for ALL requests including static files, error pages, and URLs not handled by Spring.
- Interceptors can access Spring beans directly (they run inside Spring context).
- Interceptors know which `@Controller` method will handle the request via `HandlerMethod`.
- Spring Security uses Filters (not interceptors) because security must be enforced before Spring MVC processes anything.

---

## Trap 3: HttpSession and Distributed Systems — Sticky Sessions Problem

**Trap**: "Just use `HttpSession` — it works."

**Reality**: `HttpSession` is stored in the memory of the server that created it. In a load-balanced system with multiple server instances:

- Request 1 (login) hits Server A → session created in Server A's memory → `JSESSIONID=abc` set in cookie.
- Request 2 hits Server B → Server B has no memory of session `abc` → user appears logged out.

Three solutions, in order of correctness:

**1. Sticky sessions (poor)**: Configure the load balancer to always route the same user to the same server. If Server A goes down, all its sessions are lost. Prevents horizontal scaling.

**2. Session replication (moderate)**: Servers sync session data to each other. High network overhead, complex, still problematic during node failure.

**3. Centralized session store (recommended)**: Use Spring Session with Redis. Sessions are stored in Redis, which all servers can access. Any server can handle any request.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

```java
@EnableRedisHttpSession
@Configuration
public class SessionConfig { }
// That's it — HttpSession calls now transparently use Redis
```

---

## Trap 4: CORS — Is It Client-Side or Server-Side Security?

**Trap**: "CORS is a server-side security feature that prevents unauthorized access."

**Reality**: CORS is entirely a browser-enforced mechanism. The server does not enforce CORS — the browser does. If you make the same cross-origin request using `curl`, Postman, or any non-browser tool, CORS does not apply and the request succeeds regardless of the server's CORS headers.

Implications:
- CORS does not protect your API from malicious server-to-server calls or curl attacks.
- CORS protects legitimate users from having their browser silently make cross-origin requests on their behalf (CSRF-like attacks from malicious websites).
- If your CORS configuration is wrong, it blocks your own legitimate frontend — but a determined attacker using curl is unaffected.
- Real API security requires authentication (JWT, OAuth) on the server side.

The correct mental model: CORS is a browser feature that protects users, not a server feature that protects APIs.

---

## Trap 5: Idempotency — GET, DELETE, PUT vs POST

**Trap**: "DELETE is idempotent — calling it twice gives the same result both times."

**Reality (with nuance)**: DELETE is idempotent in terms of system state, but not necessarily in terms of response code. The first `DELETE /users/1` succeeds with `204 No Content`. The second `DELETE /users/1` returns `404 Not Found`. The response codes differ — but the state of the system (no user with ID 1) is identical both times. This is the correct definition of idempotency: same final state, regardless of how many times the operation is performed.

```
Operation            Idempotent?   Safe?    Notes
─────────────────────────────────────────────────────────────────
GET                  Yes           Yes      Read-only, no side effects
HEAD                 Yes           Yes      Like GET, no body
PUT                  Yes           No       Full replacement is idempotent
DELETE               Yes           No       State same after N deletions
POST                 No            No       Each call may create new resource
PATCH                Not always    No       Depends on operation (increment vs set)
```

Safe = no server state modification. Idempotent = same result from N calls. All safe methods are idempotent, but not vice versa (DELETE is idempotent but not safe).

---

## Trap 6: HTTP 201 vs 200 for Resource Creation

**Trap**: "200 OK is fine for a successful resource creation — everyone understands it means success."

**Reality**: The HTTP specification has precise semantics. `201 Created` signals that a new resource was created as a direct result of the request, and the `Location` header must contain the URL of the new resource. Using `200` for creation violates the spec and misleads API consumers who rely on status codes programmatically.

```java
// WRONG — 200 does not convey creation semantics
@PostMapping("/users")
public User createUser(@RequestBody CreateUserRequest req) {
    return userService.create(req);  // returns 200 by default
}

// CORRECT — 201 with Location header
@PostMapping("/users")
public ResponseEntity<User> createUser(
        @RequestBody @Valid CreateUserRequest req,
        UriComponentsBuilder uriBuilder) {
    User created = userService.create(req);
    URI location = uriBuilder.path("/users/{id}").buildAndExpand(created.getId()).toUri();
    return ResponseEntity.created(location).body(created);
    // Responds: 201 Created + Location: /users/42 + body
}
```

Similarly: use `204 No Content` (not `200`) for successful DELETE and UPDATE operations where no body is needed.

---

## Trap 7: What Is the Difference Between 401 and 403?

**Trap**: "Both mean the user cannot access the resource — same thing."

**Reality**: They indicate fundamentally different situations:

- **401 Unauthorized**: "I don't know who you are. Please identify yourself." The request lacks valid authentication credentials (missing, expired, or invalid JWT/session). The client should retry with credentials. Despite the name, it really means "unauthenticated."

- **403 Forbidden**: "I know who you are, but you don't have permission." The request is properly authenticated, but the authenticated user does not have the right role or permission for this resource. Retrying with the same credentials will always fail — you need a higher-privilege account.

```
Scenario 1: No JWT token in request
→ 401 (authenticate first)

Scenario 2: Valid JWT, user is ROLE_USER, accessing /admin endpoint
→ 403 (authenticated but not authorized)

Scenario 3: Expired JWT token
→ 401 (authentication is invalid — re-authenticate)

Scenario 4: Valid JWT, user exists, but account is suspended
→ 403 (known identity, but access explicitly denied)
```

Some APIs return `404 Not Found` instead of `403` for sensitive resources to avoid revealing that the resource exists. This is a security-through-obscurity technique, not a spec violation.

---

## Trap 8: REST vs HTTP — Are They the Same?

**Trap**: "REST is just HTTP with JSON."

**Reality**: HTTP is a protocol — a standardized set of rules for transmitting data between clients and servers. REST is an architectural style — a set of constraints for designing networked APIs. REST does not require HTTP (though in practice it always uses it), and HTTP does not require REST.

A "RESTful" API must satisfy six constraints:
1. **Client-Server**: Separation of UI from data storage.
2. **Stateless**: Each request contains all information needed to process it.
3. **Cacheable**: Responses declare whether they are cacheable.
4. **Layered System**: Client cannot tell if it is connected directly to the server or via a proxy.
5. **Uniform Interface**: Resources identified in requests; manipulation through representations; self-descriptive messages; HATEOAS.
6. **Code on Demand** (optional): Server can send executable code to client.

Most so-called "REST APIs" violate several of these constraints (especially HATEOAS and statelessness when using server-side sessions). Richardson Maturity Model (levels 0–3) measures how RESTful an API actually is. Most production APIs operate at Level 2 (correct HTTP methods + status codes).

---

## Trap 9: Can GET Have a Request Body?

**Trap**: "GET requests have no body — that's a rule."

**Reality**: The HTTP specification does not prohibit a body on a GET request — it only says that the body SHOULD be ignored. In practice:
- Some servers and proxies strip the body from GET requests.
- Browsers do not send bodies with GET requests.
- Caching infrastructure (CDNs, proxies) may ignore or drop the body.
- Libraries like Elasticsearch use GET with a body for search queries (its Query DSL is complex JSON).

The general guidance is: do not use GET with a body in public APIs. The semantic convention is that GET is purely URL-parameterized. If you need to send complex query criteria, either use POST with a body (accepting the non-idempotent semantics) or encode the criteria in URL query parameters.

---

## Trap 10: @RequestParam vs @PathVariable vs @RequestBody

**Trap**: Confusing the three, or using `@RequestBody` for simple query filters.

**Reality**:

```java
// @PathVariable — part of the URL path, identifies a specific resource
@GetMapping("/users/{userId}/orders/{orderId}")
public Order getOrder(
    @PathVariable Long userId,
    @PathVariable Long orderId
) { ... }
// GET /users/42/orders/7

// @RequestParam — URL query string parameters, optional filtering/pagination
@GetMapping("/users")
public List<User> listUsers(
    @RequestParam(required = false) String role,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size
) { ... }
// GET /users?role=ADMIN&page=0&size=20

// @RequestBody — deserializes the entire HTTP request body (JSON/XML) into an object
@PostMapping("/users")
public ResponseEntity<User> createUser(
    @RequestBody @Valid CreateUserRequest req
) { ... }
// POST /users with body: {"name": "Alice", "email": "alice@example.com"}
```

Common traps:
- `@PathVariable` binds by name matching the `{variable}` in the path; name must match or use `@PathVariable("name")`.
- `@RequestParam` cannot be used to read from request body; it only reads from the URL query string and form data.
- `@RequestBody` can only be used once per method (one body per request).
- `@RequestBody` on a GET request works in Spring but violates REST conventions.

---

## Trap 11: @RequestMapping vs @GetMapping/@PostMapping

**Trap**: "I'll use `@RequestMapping` for everything — it is more generic."

**Reality**: `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, and `@PatchMapping` are convenience meta-annotations that are equivalent to `@RequestMapping(method = RequestMethod.GET)` etc. The composed annotations are more readable, type-safe, and harder to accidentally misconfigure.

```java
// Verbose and easy to forget the method attribute — defaults to ALL methods
@RequestMapping(value = "/users/{id}", method = RequestMethod.DELETE)
public ResponseEntity<Void> deleteUser(@PathVariable Long id) { ... }

// Preferred — explicit, concise
@DeleteMapping("/users/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) { ... }
```

`@RequestMapping` at the class level is still useful to define a common URL prefix:

```java
@RestController
@RequestMapping("/api/v1/users")  // prefix for all methods in this controller
public class UserController {
    @GetMapping           // GET /api/v1/users
    @GetMapping("/{id}")  // GET /api/v1/users/{id}
    @PostMapping          // POST /api/v1/users
}
```

---

## Trap 12: HTTP Caching and Cache-Control Headers

**Trap**: "Caching is handled by the frontend — the backend does not need to think about it."

**Reality**: The server controls caching behavior through response headers. Incorrect caching headers cause stale data to be served long after changes occur, or prevent caching entirely when caching would dramatically improve performance.

```java
// Spring MVC caching control
@GetMapping("/products/{id}")
public ResponseEntity<Product> getProduct(
    @PathVariable Long id,
    WebRequest request
) {
    Product product = productService.findById(id);

    // ETag-based conditional caching
    String etag = "\"" + product.getVersion() + "\"";
    if (request.checkNotModified(etag)) {
        return null; // returns 304 Not Modified automatically
    }

    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS).cachePublic())
        .eTag(etag)
        .body(product);
}

// For user-specific data — prevent public caching
@GetMapping("/users/{id}/profile")
public ResponseEntity<UserProfile> getProfile(@PathVariable Long id) {
    return ResponseEntity.ok()
        .cacheControl(CacheControl.noStore())  // never cache sensitive data
        .body(userService.getProfile(id));
}
```

Rules:
- `Cache-Control: no-store` — never cache (use for sensitive/personal data).
- `Cache-Control: no-cache` — cache but always revalidate before use.
- `Cache-Control: max-age=3600, public` — cache for 1 hour, CDN and browsers can cache.
- `Cache-Control: max-age=3600, private` — cache for 1 hour in browser only, not CDN.
