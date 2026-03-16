# Web Development — Diagram Explanations

## 1. Servlet Lifecycle

The servlet container manages the full lifecycle. The servlet is a long-lived object — created once, used for the lifetime of the application.

```
  Application Startup
         │
         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  Servlet Container loads Servlet class from web.xml or      │
  │  @WebServlet annotation                                     │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   init()        │  called ONCE
                    │  - read config  │  expensive setup here
                    │  - open conn    │  (DB pool, caches)
                    │  - load data    │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │         service()            │  called for EVERY request
              │  (called for each request)   │  runs in multiple threads
              │                              │  concurrently
              │  service() dispatches to:    │
              │  ┌──────────┐ ┌──────────┐  │
              │  │ doGet()  │ │ doPost() │  │  ← override these
              │  └──────────┘ └──────────┘  │    in your servlet
              │  ┌──────────┐ ┌──────────┐  │
              │  │ doPut()  │ │doDelete()│  │
              │  └──────────┘ └──────────┘  │
              └──────────────┬──────────────┘
                             │
                  (repeated for each request)
                             │
                  Application Shutdown
                             │
                             ▼
                    ┌─────────────────┐
                    │   destroy()     │  called ONCE
                    │  - close conn   │  cleanup here
                    │  - flush cache  │
                    └─────────────────┘
```

---

## 2. HTTP Request Flow Through Servlet Container

```
  Browser / HTTP Client
         │
         │  HTTP Request: GET /api/users/1 HTTP/1.1
         │  Host: api.example.com
         │  Accept: application/json
         ▼
  ┌────────────────────────────────────────────────────────────┐
  │              Servlet Container (Tomcat)                    │
  │                                                            │
  │  1. Accept TCP connection on port 8080                     │
  │  2. Parse HTTP headers from the byte stream                │
  │  3. Create HttpServletRequest object                       │
  │  4. Create HttpServletResponse object                      │
  │  5. Match URL to registered Servlet (URL mapping)         │
  │  6. Retrieve or create HttpSession                         │
  │                                                            │
  │  ┌─────────────────────────────────────────────────────┐  │
  │  │            Filter Chain                             │  │
  │  │  AuthFilter → LoggingFilter → CompressionFilter     │  │
  │  └─────────────────────────────────────────────────────┘  │
  │                         │                                  │
  │                         ▼                                  │
  │  ┌─────────────────────────────────────────────────────┐  │
  │  │          Servlet.service()                          │  │
  │  │  (Spring's DispatcherServlet in a Spring app)       │  │
  │  └─────────────────────────────────────────────────────┘  │
  │                         │                                  │
  │     (response written to HttpServletResponse)              │
  │                         ▼                                  │
  │  ┌─────────────────────────────────────────────────────┐  │
  │  │    Filter Chain (response direction)                │  │
  │  │  CompressionFilter → LoggingFilter → AuthFilter     │  │
  │  └─────────────────────────────────────────────────────┘  │
  │                                                            │
  │  7. Serialize HttpServletResponse to HTTP bytes            │
  │  8. Send bytes over TCP socket                             │
  └────────────────────────────────────────────────────────────┘
         │
         ▼
  Browser receives: HTTP/1.1 200 OK
                    Content-Type: application/json
                    {"id": 1, "name": "Alice"}
```

---

## 3. Filter Chain Execution

Filters execute in both directions — on the way in (request) and on the way out (response). This allows them to measure total request time, modify both request and response, and apply cross-cutting concerns uniformly.

```
  Incoming HTTP Request
         │
         ▼
  ┌──────────────────┐
  │   Filter 1       │  e.g., AuthenticationFilter
  │   pre-logic      │  checks JWT token, rejects with 401 if invalid
  └────────┬─────────┘
           │  chain.doFilter(req, res)
           ▼
  ┌──────────────────┐
  │   Filter 2       │  e.g., RequestLoggingFilter
  │   pre-logic      │  logs request URL, method, body
  └────────┬─────────┘
           │  chain.doFilter(req, res)
           ▼
  ┌──────────────────┐
  │   Filter 3       │  e.g., GzipCompressionFilter
  │   pre-logic      │  wraps response in compressing stream
  └────────┬─────────┘
           │  chain.doFilter(req, res)
           ▼
  ┌──────────────────────────────────────────┐
  │           SERVLET                        │
  │   (Spring DispatcherServlet)             │
  │   processes request, writes response     │
  └──────────────────────────┬───────────────┘
                             │  (response written)
                             ▼
  ┌──────────────────┐
  │   Filter 3       │  post-logic: finalize Gzip stream
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │   Filter 2       │  post-logic: log response status, duration
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │   Filter 1       │  post-logic: log security event if needed
  └────────┬─────────┘
           ▼
  HTTP Response sent to client

  Note: Filters run in LIFO order on the response side.
  Filter registration order determines execution order.
```

---

## 4. Spring MVC Request Lifecycle

Spring MVC routes requests through a highly configurable pipeline inside `DispatcherServlet`.

```
  HTTP Request arrives at Tomcat
         │
         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    DispatcherServlet                         │
  │                                                              │
  │  1. HandlerMapping                                           │
  │     Finds which @Controller method handles this URL         │
  │     (e.g., GET /users/{id} → UserController.getUser())      │
  │                    │                                         │
  │  2. HandlerAdapter                                           │
  │     Wraps the handler for execution;                         │
  │     applies HandlerInterceptor.preHandle()                  │
  │                    │                                         │
  │  3. Controller Method                                        │
  │     @GetMapping("/users/{id}")                               │
  │     Argument resolvers bind @PathVariable, @RequestBody      │
  │     Method executes, returns value or ResponseEntity        │
  │                    │                                         │
  │  4. ReturnValueHandler / MessageConverter                    │
  │     @ResponseBody → Jackson serializes object to JSON        │
  │     ViewResolver → resolves view name to template file       │
  │                    │                                         │
  │  5. HandlerInterceptor.postHandle()                          │
  │     (after controller, before response written)              │
  │                    │                                         │
  │  6. Response written to HttpServletResponse                  │
  │                    │                                         │
  │  7. HandlerInterceptor.afterCompletion()                     │
  │     (after response written — cleanup, metrics recording)    │
  └──────────────────────────────────────────────────────────────┘
         │
         ▼
  HTTP Response returned to client

  ExceptionHandlerExceptionResolver (runs between steps 3 and 4 on error):
  Exception → @ExceptionHandler / @ControllerAdvice → error response
```

---

## 5. Session Management

```
  Cookie-Based Session (traditional, server stores session):
  ─────────────────────────────────────────────────────────────────
  Request 1 (Login):
  Browser ──── POST /login ────────────────────────────────► Server
  Server creates session, stores data in memory or Redis:
  Sessions: { "abc123" → {userId:1, role:"ADMIN", cart:[...]} }
  Server ◄─── 200 OK                                          Server
              Set-Cookie: JSESSIONID=abc123; HttpOnly; Secure

  Request 2 (Subsequent):
  Browser ──── GET /orders                                 ► Server
               Cookie: JSESSIONID=abc123
  Server looks up "abc123" in session store, retrieves userId=1
  Server ◄─── 200 OK + orders for userId=1                   Server

  Stateless JWT-Based (modern, client stores session):
  ─────────────────────────────────────────────────────────────────
  Request 1 (Login):
  Browser ──── POST /login ────────────────────────────────► Server
  Server creates JWT: {sub:1, role:"ADMIN", exp:...} signed with secret
  Server ◄─── 200 OK + {"token": "eyJhbGci..."               Server

  Request 2 (Subsequent):
  Browser ──── GET /orders                                 ► Server
               Authorization: Bearer eyJhbGci...
  Server validates JWT signature — no session store needed
  Server ◄─── 200 OK + orders                                 Server

  Distributed Systems Problem with Server-Side Sessions:
  ─────────────────────────────────────────────────────────────────
  Load Balancer
  ┌──────┐        ► Server A (has session abc123)  ← Request 1 went here
  │Client│
  └──────┘        ► Server B (no session abc123)   ← Request 2 routed here → 401!

  Solutions:
  1. Sticky sessions (load balancer always routes to same server) — fragile
  2. Centralized session store (Redis via Spring Session) — robust
  3. Stateless JWT — no session store needed
```

---

## 6. CORS Preflight Request Flow

CORS is enforced by the browser, not the server. The server only needs to respond with the right headers.

```
  Simple Request (GET, POST with simple content-type — no preflight):
  ─────────────────────────────────────────────────────────────────
  Browser (origin: https://app.com)
    ──── GET https://api.other.com/data ───────────────────► API Server
         Origin: https://app.com
    ◄─── 200 OK ─────────────────────────────────────────── API Server
         Access-Control-Allow-Origin: https://app.com
  Browser checks header → origin allowed → delivers response to JS ✓

  Preflighted Request (PUT/DELETE, custom headers, JSON body):
  ─────────────────────────────────────────────────────────────────
  Browser detects cross-origin request requiring preflight:

  Step 1 — Browser sends preflight automatically:
    ──── OPTIONS https://api.other.com/data ────────────────► API Server
         Origin: https://app.com
         Access-Control-Request-Method: PUT
         Access-Control-Request-Headers: Content-Type, Authorization
    ◄─── 204 No Content ──────────────────────────────────── API Server
         Access-Control-Allow-Origin: https://app.com
         Access-Control-Allow-Methods: GET, POST, PUT, DELETE
         Access-Control-Allow-Headers: Content-Type, Authorization
         Access-Control-Max-Age: 3600  ← browser caches for 1 hour

  Step 2 — Browser sends actual request (only if preflight succeeded):
    ──── PUT https://api.other.com/data ────────────────────► API Server
         Origin: https://app.com
         Authorization: Bearer token
         Content-Type: application/json
    ◄─── 200 OK ─────────────────────────────────────────── API Server

  If preflight fails (wrong headers):
    ◄─── Browser blocks the actual request ──────────────────
         JS sees CORS error, never sees 401/403 from server
```

---

## 7. REST Resource Hierarchy

REST resources are organized around nouns (things), not verbs (actions). URLs form a hierarchy that reflects resource ownership.

```
  Resource Hierarchy — e-commerce example:
  ─────────────────────────────────────────────────────────────────
  /users                          ← collection of all users
  /users/{userId}                 ← single user
  /users/{userId}/orders          ← orders belonging to a user
  /users/{userId}/orders/{orderId}← specific order of a user
  /users/{userId}/orders/{orderId}/items  ← items in that order

  HTTP Methods on Resources:
  ┌─────────────────────────────────────────────────────────────┐
  │ Method    │ /users              │ /users/{id}               │
  ├───────────┼─────────────────────┼───────────────────────────┤
  │ GET       │ List all users      │ Get user by ID            │
  │ POST      │ Create new user     │ (usually 405 — use /users)│
  │ PUT       │ (usually 405)       │ Replace entire user       │
  │ PATCH     │ (usually 405)       │ Partial update            │
  │ DELETE    │ Delete all (danger!)│ Delete this user          │
  └─────────────────────────────────────────────────────────────┘

  Status codes for each operation:
  GET    /users/{id} → 200 (found) | 404 (not found)
  POST   /users      → 201 (created) + Location: /users/42
  PUT    /users/{id} → 200 (updated) | 204 (updated, no body) | 404
  PATCH  /users/{id} → 200 | 204 | 404
  DELETE /users/{id} → 204 (deleted, no body) | 404

  Actions that don't fit cleanly into CRUD:
  POST /orders/{id}/cancel          ← action as sub-resource
  POST /payments/{id}/refund        ← action on resource
  GET  /reports/monthly-summary     ← computed resource, not stored
```
