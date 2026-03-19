# Chapter 09: Web Frameworks — Diagram Explanations

## Diagram 1: Django Request/Response Lifecycle

```
Browser / Client
      │
      │  HTTP Request
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Web Server (nginx / Apache)                                    │
│  Serves static files directly; proxies dynamic requests        │
└────────────────────────────┬────────────────────────────────────┘
                             │  WSGI/ASGI
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  WSGI/ASGI Server (Gunicorn / Uvicorn)                         │
│  Manages worker processes / event loop                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  MIDDLEWARE STACK (settings.MIDDLEWARE, applied in order)      │
│                                                                 │
│  ┌─ SecurityMiddleware ────────────────────────────────────┐   │
│  │  ┌─ SessionMiddleware ──────────────────────────────┐   │   │
│  │  │  ┌─ CsrfViewMiddleware ─────────────────────┐    │   │   │
│  │  │  │  ┌─ AuthenticationMiddleware ────────┐   │    │   │   │
│  │  │  │  │                                   │   │    │   │   │
│  │  │  │  │       URL RESOLVER                │   │    │   │   │
│  │  │  │  │  (urls.py → find matching view)   │   │    │   │   │
│  │  │  │  │           │                       │   │    │   │   │
│  │  │  │  │           ▼                       │   │    │   │   │
│  │  │  │  │       VIEW FUNCTION / CBV          │   │    │   │   │
│  │  │  │  │  (queries Model, renders Template)│   │    │   │   │
│  │  │  │  └───────────────────────────────────┘   │    │   │   │
│  │  │  └──────────────────────────────────────────┘    │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │  HttpResponse
                             ▼
                      Browser / Client
```

---

## Diagram 2: FastAPI Request Lifecycle with Dependency Injection

```
HTTP Request
     │
     ▼
Uvicorn ASGI Server
     │
     ▼
FastAPI Application
     │
     ├── Middleware (CORS, GZip, custom)
     │
     ▼
Router matches path + method
     │
     ▼
Dependency Resolution (Depends graph)
     │
     ├── Depends(get_db) ────────────────► AsyncSession created
     │                                     (yields session)
     ├── Depends(get_token) ─────────────► Extract Bearer token from header
     │
     └── Depends(get_current_user)
              depends on: get_token, get_db
              ────────────────────────────► Decode JWT, load User from DB

     ▼ (all dependencies resolved)

Pydantic Validation
     ├── Path parameters
     ├── Query parameters
     ├── Request body (JSON → Pydantic model)
     └── Headers

     ▼ (validated inputs ready)

Your endpoint function runs
     │
     ├── Business logic
     ├── DB queries (using injected session)
     └── Returns Pydantic response model or dict

     ▼

Response Model Serialization
     └── Pydantic model → JSON bytes

     ▼

Response sent to client (200 OK + body)

     ▼

BackgroundTasks run (after response is sent)
     └── Email, webhooks, cleanup
```

---

## Diagram 3: N+1 Query Problem — SQL Visualization

```
BROKEN CODE: Order.objects.all()[:5], then order.customer in loop

Query 1:  SELECT id, customer_id, total FROM orders LIMIT 5;
           ┌──────┬─────────────┬────────┐
           │  id  │ customer_id │  total │
           ├──────┼─────────────┼────────┤
           │   1  │     42      │ 99.99  │
           │   2  │     17      │ 49.50  │
           │   3  │     42      │ 19.99  │
           │   4  │     88      │ 129.00 │
           │   5  │     17      │ 5.00   │
           └──────┴─────────────┴────────┘

Query 2:  SELECT * FROM customers WHERE id = 42;  ← order 1's customer
Query 3:  SELECT * FROM customers WHERE id = 17;  ← order 2's customer
Query 4:  SELECT * FROM customers WHERE id = 42;  ← DUPLICATE! order 3
Query 5:  SELECT * FROM customers WHERE id = 88;  ← order 4's customer
Query 6:  SELECT * FROM customers WHERE id = 17;  ← DUPLICATE! order 5

Total: 6 queries (1 + N). For 1000 orders: 1001 queries.


FIXED CODE: Order.objects.select_related("customer")[:5]

Query 1:  SELECT orders.*, customers.*
          FROM orders
          JOIN customers ON orders.customer_id = customers.id
          LIMIT 5;

          One query returns all needed data. No N+1.
          Django stores customer in order._state.fields_cache["customer"]
          Accessing order.customer hits the cache, no DB call.

Total: 1 query. For 1000 orders: still 1 query.
```

---

## Diagram 4: Django ORM Query Lazy Evaluation

```
QuerySet is LAZY — no SQL until evaluated

Step 1: qs = Order.objects.filter(status="pending")
             │
             │  NO SQL YET — just builds a query description
             ▼
         QuerySet object: query = "WHERE status='pending'"

Step 2: qs = qs.select_related("customer").order_by("-created_at")
             │
             │  STILL NO SQL — chaining is free, modifies the description
             ▼
         QuerySet object: query = "JOIN customers ORDER BY created_at DESC"

Step 3: qs = qs[:10]
             │
             │  STILL NO SQL — slice sets LIMIT/OFFSET on the description
             ▼
         QuerySet object: query = "... LIMIT 10"

Step 4: EVALUATION TRIGGERS SQL:
    ─── for order in qs: ──────────► SQL EXECUTED NOW
    ─── list(qs) ──────────────────► SQL EXECUTED NOW
    ─── qs[0] ─────────────────────► SQL EXECUTED NOW (LIMIT 1)
    ─── qs.count() ─────────────────► SELECT COUNT(*) (different SQL)
    ─── if qs: ─────────────────────► SQL EXECUTED (fetches first item)
    ─── bool(qs) ───────────────────► SQL EXECUTED

CACHE BEHAVIOR:
    orders = list(qs)    # SQL executed, results cached in QuerySet
    for o in qs: ...     # uses CACHE — no second SQL
    for o in qs: ...     # uses CACHE again
```

---

## Diagram 5: JWT Token Structure

```
ENCODED JWT (three base64url-encoded parts separated by dots):

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
                 .
eyJzdWIiOiI0MiIsInJvbGUiOiJ1c2VyIiwiZXhwIjoxNzA5OTk5OTk5fQ
                 .
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c


DECODED STRUCTURE:

┌──────────────────────────────────┐
│ HEADER (Algorithm & Type)        │
│ {                                │
│   "alg": "HS256",                │
│   "typ": "JWT"                   │
│ }                                │
└──────────────────────────────────┘
             .
┌──────────────────────────────────┐
│ PAYLOAD (Claims — NOT encrypted) │
│ {                                │
│   "sub": "42",       ← user id   │
│   "role": "user",                │
│   "exp": 1709999999, ← expiry    │
│   "iat": 1709996399, ← issued at │
│   "iss": "myapp"     ← issuer    │
│ }                                │
└──────────────────────────────────┘
             .
┌──────────────────────────────────┐
│ SIGNATURE                        │
│ HMAC-SHA256(                     │
│   base64(header) + "." +         │
│   base64(payload),               │
│   secret_key                     │
│ )                                │
└──────────────────────────────────┘


ACCESS vs REFRESH TOKEN FLOW:

  Client                    Auth Server              API Server
    │                           │                        │
    │── POST /login ────────────►│                        │
    │◄── access (15min) ────────│                        │
    │◄── refresh (30days) ──────│                        │
    │                           │                        │
    │── GET /api/data (access) ─────────────────────────►│
    │◄── 200 OK ────────────────────────────────────────│
    │                           │                        │
    │  [15 min later, access expired]                    │
    │── GET /api/data ───────────────────────────────────►│
    │◄── 401 Unauthorized ──────────────────────────────│
    │                           │                        │
    │── POST /refresh (refresh token) ──────────────────►auth
    │◄── new access token ──────────────────────────────│
```

---

## Diagram 6: SQLAlchemy Connection Pool

```
APPLICATION STARTUP:
  pool = create_engine(url, pool_size=5, max_overflow=10, pool_timeout=30)

                    ┌──────────────────────────────────┐
                    │        CONNECTION POOL            │
                    │                                   │
  Idle connections: │  [conn1] [conn2] [conn3] [conn4] [conn5]  │
                    │  (pool_size=5, always maintained) │
                    │                                   │
  Overflow slots:   │  [+1][+2][+3][+4][+5][+6][+7][+8][+9][+10]│
                    │  (max_overflow=10, created on demand,       │
                    │   closed when returned)            │
                    └──────────────────────────────────┘

CHECKOUT SEQUENCE:
  Request A:  pool.connect() → gets conn1 (from idle pool)
  Request B:  pool.connect() → gets conn2
  Request C:  pool.connect() → gets conn3
  Request D:  pool.connect() → gets conn4
  Request E:  pool.connect() → gets conn5 (pool now empty)
  Request F:  pool.connect() → creates overflow conn6 (+1)
  ...
  Request P:  pool.connect() → creates overflow conn15 (+10, max reached)
  Request Q:  pool.connect() → WAITS up to pool_timeout=30s
              If no conn available after 30s → TimeoutError raised

RETURN:
  Request A returns conn1 → goes back to idle pool
  Request Q immediately gets conn1

pool_recycle=3600:  connections are recycled after 1 hour
                    prevents "MySQL server has gone away" errors
                    on long-lived connections
```
