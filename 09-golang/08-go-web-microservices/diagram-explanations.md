# Go Web & Microservices — Diagram Explanations

---

## Diagram 1: Go HTTP Server Architecture

```
CLIENT (browser / curl / mobile app)
     │
     │  TCP connection (port 8080)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    net.Listener (port 8080)                      │
│         Accept() blocks waiting for new connections              │
└─────────────────────────────────────────────────────────────────┘
     │
     │  New connection accepted
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  go conn.serve()  — new goroutine per connection                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Read HTTP request from connection                         │  │
│  │  Build *http.Request                                       │  │
│  │  Create http.ResponseWriter                                │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
     │
     │  Call Handler.ServeHTTP(w, r)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│                     http.ServeMux                                │
│  Pattern matching:                                               │
│  "GET /users/{id}"  ──► getUserHandler                          │
│  "POST /users"      ──► createUserHandler                       │
│  "GET /health"      ──► healthHandler                           │
│  "/"                ──► notFoundHandler (catch-all)             │
└─────────────────────────────────────────────────────────────────┘
     │
     │  Route matched
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Middleware Chain (each wraps the next):                         │
│                                                                  │
│  ┌──────────────┐                                               │
│  │  RequestID   │  injects X-Request-ID into context            │
│  │  ┌──────────────────────────────────────────────────────┐   │
│  │  │  Logging   │  records method, path, status, latency  │   │
│  │  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │  │  Recovery  │  catches panics, returns 500     │  │   │
│  │  │  │  ┌──────────────────────────────────────────┐  │  │   │
│  │  │  │  │  Auth      │  validates JWT token      │  │  │   │
│  │  │  │  │  ┌──────────────────────────────────┐   │  │  │   │
│  │  │  │  │  │  Handler  (your business logic)  │   │  │  │   │
│  │  │  │  │  └──────────────────────────────────┘   │  │  │   │
│  │  │  │  └──────────────────────────────────────────┘  │  │   │
│  │  │  └────────────────────────────────────────────────┘  │   │
│  │  └──────────────────────────────────────────────────────┘   │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
     │
     │  Write response back to TCP connection
     ▼
CLIENT receives HTTP response
```

---

## Diagram 2: Middleware Chain (Onion Model)

```
                    HTTP Request enters here
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: RequestID Middleware                                   │
│  BEFORE: generate/read X-Request-ID, add to context             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  LAYER 2: Logging Middleware                               │  │
│  │  BEFORE: record start time                                 │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  LAYER 3: Recovery Middleware                         │  │  │
│  │  │  BEFORE: set up deferred recover()                    │  │  │
│  │  │  ┌───────────────────────────────────────────────┐   │  │  │
│  │  │  │  LAYER 4: Auth Middleware                      │   │  │  │
│  │  │  │  BEFORE: validate JWT, add claims to context   │   │  │  │
│  │  │  │  ┌─────────────────────────────────────────┐  │   │  │  │
│  │  │  │  │  YOUR HANDLER                            │  │   │  │  │
│  │  │  │  │  decode request → business logic → write │  │   │  │  │
│  │  │  │  └─────────────────────────────────────────┘  │   │  │  │
│  │  │  │  AFTER: nothing (auth doesn't modify response) │   │  │  │
│  │  │  └───────────────────────────────────────────────┘   │  │  │
│  │  │  AFTER: recover from panic → write 500               │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │  AFTER: log status code, latency, bytes                    │  │
│  └───────────────────────────────────────────────────────────┘  │
│  AFTER: nothing more to do                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                    HTTP Response exits here

Request traversal order:  1 → 2 → 3 → 4 → handler
Response traversal order: handler → 4 → 3 → 2 → 1

NOTE: Code that runs BEFORE next.ServeHTTP() sees the request.
      Code that runs AFTER next.ServeHTTP() sees the response.
```

---

## Diagram 3: Graceful Shutdown Signal Flow

```
RUNNING STATE:
┌─────────────────────────────────────────────────────────────────┐
│  main goroutine                                                  │
│  signal.NotifyContext(ctx, SIGINT, SIGTERM) ─► blocks on <-ctx  │
│                                                                  │
│  HTTP server goroutine                                           │
│  srv.ListenAndServe() ─► accepting connections                   │
│                                                                  │
│  Worker goroutines (active requests):                            │
│  goroutine-1: handling POST /orders  [processing...]             │
│  goroutine-2: handling GET /users/5  [processing...]             │
│  goroutine-3: handling PUT /items/3  [processing...]             │
└─────────────────────────────────────────────────────────────────┘

    ↓ SIGTERM received from OS (e.g., kubectl delete pod)

SHUTDOWN INITIATED:
┌─────────────────────────────────────────────────────────────────┐
│  Step 1: ctx is cancelled                                        │
│          main goroutine unblocks                                  │
│                                                                  │
│  Step 2: srv.Shutdown(ctx) called                                │
│          ┌─────────────────────────────────────────────────┐    │
│          │ TCP listener CLOSED — no new connections accepted│    │
│          └─────────────────────────────────────────────────┘    │
│                                                                  │
│  Step 3: Wait for in-flight requests (up to 30s):               │
│  goroutine-1: handling POST /orders  [5s... done ✓]             │
│  goroutine-2: handling GET /users/5  [100ms... done ✓]           │
│  goroutine-3: handling PUT /items/3  [2s... done ✓]             │
│                                                                  │
│  Step 4: All handlers returned — srv.Shutdown() returns nil      │
│                                                                  │
│  Step 5: Run cleanup hooks:                                      │
│          db.Close()        ← drain connection pool               │
│          cache.Close()     ← flush pending cache writes          │
│          metrics.Flush()   ← send final metrics                  │
│                                                                  │
│  Step 6: main() returns — process exits with code 0             │
└─────────────────────────────────────────────────────────────────┘

TIMEOUT SCENARIO (request takes > 30s):
│  Step 3 (continued): goroutine-1 still running after 30s        │
│          Context deadline exceeded!                               │
│          srv.Close() called — connections forcibly closed         │
│          goroutine-1 receives context cancellation               │
│          Process exits with non-zero (or log warning)            │
```

---

## Diagram 4: gRPC vs REST Comparison

```
REST/JSON over HTTP/1.1:
─────────────────────────────────────────────────────────────────
Client                                              Server
  │                                                    │
  │── TCP SYN ─────────────────────────────────────► │
  │◄─ TCP SYN-ACK ──────────────────────────────────  │
  │── TCP ACK ─────────────────────────────────────► │
  │   (3-way handshake complete)                       │
  │                                                    │
  │── POST /users HTTP/1.1 ────────────────────────► │  ← text header
  │   Content-Type: application/json                  │
  │   {"name":"Alice","email":"a@b.com","age":30}    │  ← text body (36 bytes)
  │                                                    │
  │◄─ HTTP/1.1 201 Created ─────────────────────────  │  ← text header
  │   Content-Type: application/json                  │
  │   {"id":"123","name":"Alice","email":"a@b.com"}   │  ← text body (44 bytes)
  │                                                    │
  │  [connection closes or waits idle]                │

gRPC/Protobuf over HTTP/2:
─────────────────────────────────────────────────────────────────
Client                                              Server
  │                                                    │
  │── TCP + TLS handshake ─────────────────────────► │
  │── HTTP/2 SETTINGS frame ───────────────────────► │  ← single connection established
  │                                                    │  (reused for ALL requests!)
  │── HEADERS frame (stream 1) ───────────────────► │  ← binary, compressed
  │── DATA frame: [0a 05 41 6c 69 63 65 11 ...] ──► │  ← protobuf: 11 bytes vs 36 JSON
  │                                                    │
  │── HEADERS frame (stream 3) ───────────────────► │  ← CONCURRENT on same TCP conn!
  │── DATA frame: [0a 03 42 6f 62 ...] ────────────► │
  │                                                    │
  │◄─ HEADERS frame (stream 1) ─────────────────────  │
  │◄─ DATA frame: [0a 03 31 32 33 ...] ─────────────  │  ← protobuf: ~8 bytes vs 44 JSON
  │                                                    │
  │◄─ HEADERS frame (stream 3) ─────────────────────  │  ← stream 3 response arrives
  │◄─ DATA frame: ...                                 │


PERFORMANCE COMPARISON:
┌─────────────────┬──────────────────┬──────────────────────────┐
│ Metric          │ REST/JSON        │ gRPC/Protobuf             │
├─────────────────┼──────────────────┼──────────────────────────┤
│ Serialization   │ ~500 ns          │ ~50 ns (10× faster)       │
│ Payload size    │ 100 bytes        │ ~30 bytes (3× smaller)    │
│ Connections     │ 1 per request    │ 1 per service pair        │
│ Streaming       │ SSE only         │ native bidirectional      │
│ Type safety     │ runtime          │ compile-time              │
│ Browser support │ native           │ needs grpc-web            │
│ Debugging       │ curl/Postman     │ grpcurl/Postman gRPC      │
└─────────────────┴──────────────────┴──────────────────────────┘
```

---

## Diagram 5: Microservice with Health Checks, Metrics, Tracing

```
                        EXTERNAL TRAFFIC
                              │
                              ▼
                    ┌─────────────────┐
                    │  Load Balancer   │
                    │                  │
                    │ Checks /readyz   │  ← removes unhealthy pods
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Your Service  │
                    │                  │
                    │  :8080 (API)    │  ← handles business requests
                    │  :8081 (Admin)  │  ← internal only (not exposed)
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │ /healthz     │ │ /metrics     │ │ /debug/pprof │
     │              │ │              │ │              │
     │ Liveness:    │ │ Prometheus   │ │ Go pprof     │
     │ "alive?"     │ │ metrics:     │ │ profiling    │
     │              │ │ - http_reqs  │ │              │
     │ Kubernetes   │ │ - gc_pauses  │ │ Only in non- │
     │ restarts if  │ │ - goroutines │ │ production   │
     │ this fails   │ │              │ │              │
     └──────────────┘ └──────────────┘ └──────────────┘

     ┌──────────────┐
     │ /readyz      │
     │              │
     │ Readiness:   │
     │ checks:      │
     │ - DB ping    │
     │ - Redis ping │
     │ - Disk space │
     │              │
     │ LB removes   │
     │ if unhealthy │
     └──────────────┘

DISTRIBUTED TRACING FLOW:
┌──────────────┐    trace-id: abc123     ┌──────────────────┐
│  API Gateway │  ───────────────────►   │  Order Service   │
│  span: root  │                         │  span: child 1   │
└──────────────┘                         └────────┬─────────┘
                                                  │ trace-id: abc123
                                         ┌────────▼─────────┐
                                         │ Inventory Service │
                                         │ span: child 2     │
                                         └──────────────────┘

All spans collected by OpenTelemetry Collector → Jaeger/Zipkin UI
Trace shows: Gateway (100ms) → Order (80ms) → Inventory (60ms)
```

---

## Diagram 6: Circuit Breaker State Machine

```
STATES:
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│   ┌──────────┐                              ┌─────────────┐  │
│   │          │                              │             │  │
│   │  CLOSED  │ ◄──────── SUCCESS ──────────│  HALF-OPEN  │  │
│   │ (normal) │                              │  (probing)  │  │
│   │          │                              │             │  │
│   └────┬─────┘                              └──────┬──────┘  │
│        │                                           │         │
│        │ N consecutive failures                    │ FAILURE  │
│        │ (e.g., 5 timeouts)                        │          │
│        ▼                                           ▼         │
│   ┌────────────────────────────────────────────────────────┐ │
│   │                       OPEN                             │ │
│   │              (fast-failing — no calls made)            │ │
│   └────────────────────────┬───────────────────────────────┘ │
│                             │                                 │
│                    Reset timeout expires                       │
│                    (e.g., 30 seconds)                         │
│                             │                                 │
│                             └──────────────► HALF-OPEN       │
│                                                               │
└───────────────────────────────────────────────────────────────┘

BEHAVIOR IN EACH STATE:

CLOSED (normal):
  Request ──► downstream service ──► success
  Request ──► downstream service ──► failure (count++)
  After 5 failures: transition to OPEN

  ┌────┐  req   ┌─────────────────┐  call   ┌──────────────┐
  │    │ ─────► │ Circuit Breaker │ ──────► │ External API │
  │    │ ◄───── │ (CLOSED state)  │ ◄────── │              │
  └────┘  resp  └─────────────────┘  resp   └──────────────┘

OPEN (fast-fail):
  Request ──► circuit breaker ──► ErrCircuitOpen (immediate, no network call)

  ┌────┐  req   ┌─────────────────┐  FAST   ╳──────────────╳
  │    │ ─────► │ Circuit Breaker │ FAIL!   │ NOT called!   │
  │    │ ◄───── │ (OPEN state)    │         │               │
  └────┘  err   └─────────────────┘         ╳──────────────╳

HALF-OPEN (testing):
  1 probe request allowed ──► if success: CLOSE; if failure: OPEN

  ┌────┐  req   ┌─────────────────┐  probe  ┌──────────────┐
  │    │ ─────► │ Circuit Breaker │ ──────► │ External API │
  │    │        │ (HALF-OPEN)     │         │ (recovered?) │
  └────┘        └─────────────────┘         └──────────────┘
                        │                          │
                   failure?                    success?
                        │                          │
                        ▼                          ▼
                     OPEN                        CLOSED
                (stay open 30s more)         (resume normal)

TIMELINE EXAMPLE:
t=0s:   5 failures in 10s ───────────────────► OPEN
t=0-30s: all requests fast-fail (returns immediately)
t=30s:  timeout expires ─────────────────────► HALF-OPEN
t=30s:  1 probe request ─────────────────────► success → CLOSED
t=31s:  normal operation resumes
```
