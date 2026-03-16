# Web Development — Structured Answers

## 1. What Is the Servlet Lifecycle?

A Servlet is a Java class managed by a Servlet container (Tomcat, Jetty, Undertow). The container controls the complete lifecycle through three methods:

**`init(ServletConfig config)`** — called once when the container instantiates the servlet. Expensive initialization (opening database connections, loading configuration, initializing caches) belongs here. If `load-on-startup` is set to a non-negative integer in `web.xml` or `@WebServlet(loadOnStartup=1)`, initialization happens at application startup rather than on the first request.

**`service(HttpServletRequest req, HttpServletResponse res)`** — called once per HTTP request, potentially from many concurrent threads. The default `HttpServlet.service()` implementation dispatches to `doGet()`, `doPost()`, `doPut()`, `doDelete()`, etc. based on the HTTP method. This method must be thread-safe because a single servlet instance handles all concurrent requests.

**`destroy()`** — called once when the container is shutting down or the servlet is being removed. Use it to release resources: close database connections, flush buffers, stop background threads.

```java
@WebServlet(urlPatterns = "/api/status", loadOnStartup = 1)
public class StatusServlet extends HttpServlet {

    private HealthChecker healthChecker;

    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
        String dbUrl = config.getInitParameter("dbUrl");
        this.healthChecker = new HealthChecker(dbUrl);  // expensive setup — once
    }

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        // Use only local variables — thread-safe
        String status = healthChecker.check();           // stateless call
        resp.setContentType("application/json");
        resp.getWriter().write("{\"status\":\"" + status + "\"}");
    }

    @Override
    public void destroy() {
        healthChecker.close();  // cleanup — once on shutdown
    }
}
```

The critical thread-safety point: the container creates one servlet instance and calls `service()` from multiple threads simultaneously. Instance variables must be immutable or thread-safe. In practice, Spring's `DispatcherServlet` follows this same contract — all `@Controller` beans are singletons and must be stateless with respect to individual request data.

---

## 2. What Is the Difference Between a Filter and a Spring Interceptor?

Both are mechanisms to apply cross-cutting concerns (logging, authentication, CORS) to HTTP requests, but they operate at different layers of the stack and have different capabilities.

**Servlet Filter** (`javax.servlet.Filter`):
- Part of the Servlet specification, not Spring-specific.
- Managed by the Servlet container (Tomcat).
- Executes before Spring MVC processes the request.
- Runs for ALL requests: static files, error pages, actuator endpoints, and Spring-managed endpoints alike.
- Has access to raw `HttpServletRequest` and `HttpServletResponse`.
- Cannot access Spring beans directly (unless Spring's `DelegatingFilterProxy` is used).
- Can completely replace or wrap the request/response (useful for reading request body twice, adding headers, GZIP).

**Spring HandlerInterceptor** (`org.springframework.web.servlet.HandlerInterceptor`):
- Spring MVC-specific concept, runs inside `DispatcherServlet`.
- Executes after the Servlet container's filter chain.
- Only runs for requests matched by a `HandlerMapping` (i.e., requests handled by `@Controller` methods).
- Has access to the `HandlerMethod` — knows which controller and method will handle the request.
- Full access to the Spring application context.
- Three hook points: `preHandle()` (before controller), `postHandle()` (after controller, before response write), `afterCompletion()` (after response sent, for cleanup/logging).

```java
// Filter — runs before Spring, wraps request/response
@Component
public class RequestTimingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        long start = System.currentTimeMillis();
        chain.doFilter(req, res);  // proceed through filter chain and servlet
        long duration = System.currentTimeMillis() - start;
        log.info("Request took {}ms", duration);
    }
}

// Interceptor — runs inside Spring, knows the target handler
@Component
public class AuditInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        if (handler instanceof HandlerMethod hm) {
            String controllerName = hm.getBeanType().getSimpleName();
            String methodName = hm.getMethod().getName();
            auditLog.info("Accessing {}.{}", controllerName, methodName);
        }
        return true; // return false to abort request processing
    }

    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res,
            Object handler, Exception ex) {
        MDC.clear(); // clean up request-scoped logging context
    }
}
```

**When to use which**:
- Spring Security: Filter (must run before Spring sees the request).
- CORS headers: Filter (must apply to all requests including preflight OPTIONS).
- Request/response body logging: Filter (body can only be read once without a wrapper).
- Role-based access to specific controllers: Interceptor (knows the HandlerMethod).
- Populating thread-local request context: either, but Filter is more universal.

---

## 3. How Does Session Management Work in a Distributed Environment?

In a single-server deployment, `HttpSession` works by storing session data in the server's JVM memory, keyed by a session ID that is sent to the client as a `JSESSIONID` cookie. On subsequent requests, the cookie is sent back, and the server looks up the session.

This breaks in a distributed environment because session data lives in one server's memory. When a load balancer routes a subsequent request to a different server, that server has no knowledge of the session.

**Solution 1 — Sticky Sessions (poor)**:
Configure the load balancer to always route requests with a given `JSESSIONID` to the same backend server. This works until that server fails or restarts, at which point all its sessions are lost. It also prevents even load distribution and prevents auto-scaling.

**Solution 2 — Spring Session with Redis (recommended)**:
Spring Session transparently replaces the default `HttpSession` implementation with one backed by Redis. All servers share the same Redis instance, so any server can handle any request.

```xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
</dependency>
```

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
    // Spring Session auto-configures with spring.data.redis.host/port
}

// Usage is identical to regular HttpSession — no code changes needed
@RestController
public class CartController {
    @PostMapping("/cart/add")
    public void addItem(HttpSession session, @RequestBody CartItem item) {
        List<CartItem> cart = (List<CartItem>) session.getAttribute("cart");
        if (cart == null) cart = new ArrayList<>();
        cart.add(item);
        session.setAttribute("cart", cart);  // stored in Redis transparently
    }
}
```

**Solution 3 — JWT (stateless, no session store)**:
The server issues a signed JWT on login. The client stores it (localStorage or HTTP-only cookie) and sends it on every request. The server validates the signature — no shared state required. Enables true horizontal scaling but requires token revocation handling (maintain a blacklist in Redis for logout).

---

## 4. What Is CORS and Why Does It Exist?

CORS (Cross-Origin Resource Sharing) is a browser security mechanism that controls whether JavaScript code running on one origin (protocol + domain + port) can make HTTP requests to a different origin.

**Why it exists**: Browsers enforce the same-origin policy — JavaScript on `https://evil.com` cannot read responses from `https://your-bank.com`. Without this, a malicious website could use your browser (and your active session cookies) to silently call your bank's API and exfiltrate your data.

CORS is the mechanism by which a server can explicitly grant permission to specific origins to make cross-origin requests.

**How it works**: For "preflighted" requests (non-simple requests — those using PUT/DELETE, custom headers, or JSON body), the browser first sends an `OPTIONS` preflight request. The server responds with the origins, methods, and headers it permits. The browser then either allows or blocks the actual request based on that response.

**Spring Boot CORS configuration**:

```java
// Method 1 — Fine-grained control with @CrossOrigin
@RestController
@CrossOrigin(origins = "https://app.example.com", maxAge = 3600)
public class UserController {
    @GetMapping("/users")
    public List<User> list() { ... }
}

// Method 2 — Global configuration (recommended for production)
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com", "https://admin.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .allowedHeaders("Authorization", "Content-Type", "X-Requested-With")
            .allowCredentials(true)   // required for cookies/Authorization header
            .maxAge(3600);            // cache preflight response for 1 hour
    }
}

// Method 3 — Spring Security integration (when using Spring Security)
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
    // ...
}

@Bean
CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowCredentials(true);
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

Important: `allowedOrigins("*")` cannot be combined with `allowCredentials(true)`. The spec prohibits wildcards when credentials (cookies, Authorization headers) are involved. Use `allowedOriginPatterns("https://*.example.com")` instead.

---

## 5. What Makes an API Truly RESTful?

REST (Representational State Transfer) is an architectural style defined by Roy Fielding in his 2000 dissertation. A truly RESTful API satisfies six constraints:

**1. Stateless**: Every request must contain all information needed to process it. The server stores no client session state between requests. Authentication must be resent with every request (JWT, API key). This enables horizontal scaling — any server can handle any request.

**2. Client-Server Separation**: The UI (client) is decoupled from the data store (server). They evolve independently. A mobile app, web app, and CLI can all use the same API.

**3. Cacheable**: Responses must declare whether they are cacheable. `Cache-Control: max-age=300` allows CDNs and browsers to cache. Cacheable responses reduce server load.

**4. Uniform Interface**: This is the core of REST, with four sub-constraints:
   - Resource identification: resources identified by URI (`/users/42`)
   - Manipulation through representations: client manipulates resources by sending representations (JSON/XML)
   - Self-descriptive messages: each message includes enough information to describe how to process it (Content-Type, HTTP method)
   - HATEOAS: responses include hyperlinks to related actions (rarely implemented in practice)

**5. Layered System**: Client cannot distinguish between a direct connection and one through a proxy, CDN, or load balancer.

**6. Code on Demand** (optional): Server can send executable code (JavaScript) to client.

```java
// Level 0 REST (RPC over HTTP — not REST)
POST /userService?action=getUser&id=42

// Level 1 REST (Resources, but wrong methods)
POST /users/42/get

// Level 2 REST (Correct HTTP methods + status codes — most production APIs)
GET    /users/42           → 200 User
POST   /users              → 201 Created + Location: /users/42
PUT    /users/42           → 200 Updated
DELETE /users/42           → 204 No Content

// Level 3 REST (HATEOAS — true REST, rarely seen)
GET /users/42
Response:
{
  "id": 42,
  "name": "Alice",
  "_links": {
    "self":   {"href": "/users/42"},
    "orders": {"href": "/users/42/orders"},
    "delete": {"href": "/users/42", "method": "DELETE"}
  }
}
```

Most production APIs operate at Richardson Maturity Level 2. True REST (Level 3 with HATEOAS) is rare outside of academic implementations.

---

## 6. How Do You Implement Content Negotiation in Spring Boot?

Content negotiation allows a single endpoint to return different representations of the same resource based on the client's `Accept` header.

```java
@RestController
@RequestMapping("/reports")
public class ReportController {

    @GetMapping(value = "/{id}",
        produces = { MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE })
    public Report getReport(@PathVariable Long id) {
        return reportService.findById(id);
        // Jackson serializes to JSON if Accept: application/json
        // Jackson XML extension serializes to XML if Accept: application/xml
    }
}
```

For custom media types (e.g., CSV export):

```java
@GetMapping(value = "/export", produces = "text/csv")
public ResponseEntity<Resource> exportCsv() {
    byte[] csv = reportService.generateCsv();
    return ResponseEntity.ok()
        .contentType(MediaType.parseMediaType("text/csv"))
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"report.csv\"")
        .body(new ByteArrayResource(csv));
}

// Custom HttpMessageConverter for CSV
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new CsvMessageConverter());
    }
}
```

**Request content negotiation** (what the client is sending) is indicated by `Content-Type`. The `@RequestMapping(consumes = ...)` attribute restricts which content types a method accepts:

```java
@PostMapping(
    value = "/users",
    consumes = MediaType.APPLICATION_JSON_VALUE,       // only accept JSON body
    produces = MediaType.APPLICATION_JSON_VALUE        // always respond with JSON
)
public ResponseEntity<User> createUser(@RequestBody @Valid CreateUserRequest req) { ... }
```

If the client sends `Accept: application/xml` but the server can only produce JSON, Spring returns `406 Not Acceptable`.

---

## 7. What Is the Difference Between HTTP 401 and 403?

**401 Unauthorized** (misleadingly named — it actually means Unauthenticated):
The request lacks valid authentication credentials. The server does not know who is making the request. The response should include a `WWW-Authenticate` header indicating the authentication scheme required. The client should retry with credentials.

**403 Forbidden**:
The request is properly authenticated — the server knows who you are — but you are not permitted to access the requested resource. Retrying with the same credentials will always produce the same result. The client needs different credentials (a different account with higher privileges).

```java
// Spring Security maps these correctly:
// No token / expired token → 401
// Valid token, insufficient role → 403

@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, authEx) ->
                    res.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Authentication required")
                )
                .accessDeniedHandler((req, res, accessEx) ->
                    res.sendError(HttpServletResponse.SC_FORBIDDEN, "Insufficient privileges")
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );
        return http.build();
    }
}
```

Security note: some APIs deliberately return `404 Not Found` instead of `403` to avoid leaking information about the existence of a protected resource to unauthenticated callers. This is a valid security technique but should be a conscious architectural decision, not an accident.

---

## 8. How Does Spring MVC's DispatcherServlet Handle a Request?

`DispatcherServlet` is the Front Controller for all Spring MVC request processing. It is the single entry point that routes requests to the appropriate handler.

```java
// Step-by-step processing flow:

// 1. DispatcherServlet receives HTTP request from Servlet container

// 2. HandlerMapping — find which handler matches this request
//    RequestMappingHandlerMapping scans @RequestMapping annotations
//    Returns HandlerExecutionChain (handler + interceptors)
//    e.g., GET /users/42 → UserController.getUser(Long id)

// 3. HandlerAdapter — delegate execution to the right adapter
//    RequestMappingHandlerAdapter executes @RequestMapping methods
//    Invokes HandlerInterceptor.preHandle() for each interceptor

// 4. Argument Resolution — prepare method parameters
//    @PathVariable → extracted from URI template
//    @RequestBody → Jackson deserializes JSON to object
//    @RequestParam → extracted from query string
//    HttpServletRequest/Response → directly injected

// 5. Controller method executes
//    Business logic runs, service layer called

// 6. Return value handling
//    @ResponseBody → HttpMessageConverter serializes return value
//    ResponseEntity → status, headers, body extracted
//    String view name → forwarded to ViewResolver

// 7. HandlerInterceptor.postHandle() called (before response written)

// 8. Response written to HttpServletResponse

// 9. HandlerInterceptor.afterCompletion() called (after response written)

// On exception at any step:
// HandlerExceptionResolver chain processes the exception
// @ExceptionHandler in @ControllerAdvice → custom error response
```

```java
// Custom exception handling via @ControllerAdvice
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(EntityNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return new ErrorResponse("VALIDATION_FAILED", message);
    }
}
```

---

## 9. What Is Idempotency and Why Does It Matter in API Design?

An HTTP operation is idempotent if executing it N times produces the same server state as executing it once. This property is critical for reliable distributed systems because networks are unreliable — requests can be lost, duplicated, or timeout in transit.

**Why it matters**: If a client sends a `DELETE /orders/42` request and the network times out before receiving a response, the client does not know if the server processed the request. Because `DELETE` is idempotent, the client can safely retry — if the first request succeeded, the second gets a `404`, but the state of the system is correct. If `DELETE` were not idempotent, retrying could delete a different resource or cause unintended side effects.

**POST and idempotency**: `POST` is not idempotent by definition. If the client retries a `POST /orders` request after a timeout, a duplicate order might be created. Solutions:

```java
// Idempotency key — client generates a unique key for each logical operation
// Server stores the key and returns the same result for duplicate requests
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(
        @RequestBody CreateOrderRequest req,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {

    // Check if we've seen this key before
    Optional<Order> existing = idempotencyStore.findByKey(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(existing.get()); // return same result as before
    }

    Order order = orderService.create(req);
    idempotencyStore.save(idempotencyKey, order, Duration.ofHours(24));
    return ResponseEntity.created(location(order)).body(order);
}
```

**PUT and PATCH idempotency**:
- `PUT` with a full resource representation is idempotent — `PUT /users/1` with `{"name":"Alice","email":"alice@x.com"}` always leaves the same state.
- `PATCH` may or may not be idempotent depending on the operation: `{"op":"set","path":"/name","value":"Alice"}` is idempotent; `{"op":"increment","path":"/viewCount"}` is not.

---

## 10. How Do You Implement Request/Response Logging in Spring Boot?

Comprehensive request/response logging requires a Servlet Filter because the body of an HTTP request can only be read once from the stream. Reading it in a filter requires wrapping the request.

```java
// ContentCachingRequestWrapper caches the body so it can be read multiple times
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestResponseLoggingFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(RequestResponseLoggingFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
            FilterChain chain) throws ServletException, IOException {

        ContentCachingRequestWrapper wrappedReq = new ContentCachingRequestWrapper(req);
        ContentCachingResponseWrapper wrappedRes = new ContentCachingResponseWrapper(res);

        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);

        try {
            chain.doFilter(wrappedReq, wrappedRes);
        } finally {
            long duration = System.currentTimeMillis() - startTime;

            String requestBody = new String(wrappedReq.getContentAsByteArray(),
                StandardCharsets.UTF_8);
            String responseBody = new String(wrappedRes.getContentAsByteArray(),
                StandardCharsets.UTF_8);

            log.info("REQUEST  [{}] {} {} | body: {}",
                requestId, req.getMethod(), req.getRequestURI(),
                truncate(requestBody, 500));
            log.info("RESPONSE [{}] {} {}ms | body: {}",
                requestId, res.getStatus(), duration,
                truncate(responseBody, 500));

            wrappedRes.copyBodyToResponse(); // IMPORTANT: write response body to actual response
            MDC.clear();
        }
    }

    private String truncate(String s, int maxLen) {
        return s.length() > maxLen ? s.substring(0, maxLen) + "..." : s;
    }
}
```

For structured logging with correlation IDs (for distributed tracing):

```java
// Populate MDC with trace ID from incoming request (e.g., from a gateway)
MDC.put("traceId", req.getHeader("X-Trace-ID"));
MDC.put("spanId", UUID.randomUUID().toString().substring(0, 8));
// logback.xml pattern: %X{traceId} %X{spanId} %msg
```

For production, consider using Spring Boot Actuator's `HttpExchangesAutoConfiguration` or the `spring-boot-starter-actuator` which provides `/actuator/httpexchanges` endpoint to view recent HTTP exchanges without writing custom filter code.
