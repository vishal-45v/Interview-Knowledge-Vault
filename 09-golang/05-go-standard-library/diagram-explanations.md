# Go Standard Library — Diagram Explanations

## Diagram 1: net/http Server Request Flow

```
INCOMING TCP CONNECTION
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  net.Listener (bound to :8080)                                  │
│  Accept() → net.Conn                                           │
└─────────────────────────────────────────────────────────────────┘
         │ one goroutine spawned per connection
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  http.Server (reads request, writes response)                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ReadTimeout   ────────────────────────────►            │   │
│  │  ReadHeaderTimeout ──────────►                          │   │
│  │  WriteTimeout         ──────────────────────────────►   │   │
│  │  IdleTimeout (between keep-alive requests)              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Parses HTTP request → *http.Request                           │
└─────────────────────────────────────────────────────────────────┘
         │ passes (ResponseWriter, *Request)
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Handler (the outermost)                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Middleware chain (each calls next.ServeHTTP)           │   │
│  │                                                          │   │
│  │  RecoveryMiddleware                                      │   │
│  │    └─► LoggerMiddleware                                  │   │
│  │          └─► AuthMiddleware                              │   │
│  │                └─► http.ServeMux (router)                │   │
│  │                       │                                  │   │
│  │                       ├─ "/health"  → healthHandler      │   │
│  │                       ├─ "/api/"   → apiHandler          │   │
│  │                       └─ "/"       → 404 handler         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
         │ handler writes to http.ResponseWriter
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  http.ResponseWriter                                            │
│  w.Header().Set(...)    → buffered headers                     │
│  w.WriteHeader(200)     → sends status line + headers          │
│  w.Write(body)          → sends response body                  │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
  TCP connection (reused for keep-alive, or closed)

─────────────────────────────────────────────────────────────────

IMPORTANT TIMING:

  Client connects
       │
  ReadHeaderTimeout starts ──────────────────────┐
       │                                           │
  Request headers arrive                         (if timeout: 408)
       │
  ReadTimeout continues ─────────────────────────┐
       │                                           │
  Request body fully read                        (if timeout: 408)
       │
  Handler executes
       │
  WriteTimeout starts ───────────────────────────┐
       │                                           │
  Response fully sent                            (if timeout: connection closed)
```

---

## Diagram 2: io.Reader Composition Chain

```
READING FROM A GZIPPED, ENCRYPTED HTTP RESPONSE AND COMPUTING CHECKSUM:

  resp.Body (io.ReadCloser)
       │  ← net/http reads from TCP socket
       ▼
  io.LimitReader(resp.Body, 10MB)
       │  ← enforces maximum size
       ▼
  gzip.NewReader(limited)
       │  ← decompresses on-the-fly
       ▼
  cipher.StreamReader{cipher, gzip}
       │  ← decrypts on-the-fly
       ▼
  io.TeeReader(decrypted, hashWriter)
       │  ← copies bytes to SHA256 hasher simultaneously
       ▼
  bufio.NewReader(tee)
       │  ← buffers for efficient line-by-line reading
       ▼
  bufio.Scanner(buffered)
       │
       ▼
  scanner.Scan() → line by line processing

Each layer in the chain:
  - Implements io.Reader
  - Reads from the previous layer
  - Transforms the data in some way
  - Passes transformed bytes to the next layer

No layer knows about any other layer — only the Reader interface.

─────────────────────────────────────────────────────────────────

COMMON io UTILITY FUNCTIONS:

io.Copy(dst, src)         src ──────────────────────► dst
io.TeeReader(src, w)      src ──────────► consumer
                                └───────► w (simultaneous copy)
io.MultiReader(r1,r2,r3)  r1 → r2 → r3 ─────────────► reader (sequential)
io.MultiWriter(w1,w2,w3)  writer ──────────────────── ► w1, w2, w3 (fan-out)
io.LimitReader(r, n)      r (stops after n bytes) ───► reader
io.Pipe()                 pr ◄──── (pipe) ───────────── pw
                          (synchronous, in-memory)
```

---

## Diagram 3: HTTP Client Connection Pool and Reuse

```
FIRST REQUEST to api.example.com:

  client.Do(req)
       │
       ▼
  Transport checks idle pool for api.example.com:443
       │
  Pool is EMPTY → dial new TCP connection
       │
  TLS handshake (TLSHandshakeTimeout)
       │
  Connection established
       │
  Send HTTP request
       │
  Receive response headers (ResponseHeaderTimeout)
       │
  Return *http.Response to caller
       │
  Caller reads resp.Body
       │
  resp.Body.Close() + io.Discard drain
       │
       ▼
  Connection returns to idle pool ──► [conn1]

─────────────────────────────────────────────────────────────────

SECOND REQUEST to same host:

  client.Do(req2)
       │
       ▼
  Transport checks idle pool for api.example.com:443
       │
  Pool has [conn1] → REUSE
       │
  Send HTTP request (no TCP dial, no TLS)
       │
  Receive response
       │
  resp.Body.Close()
       │
       ▼
  Connection returns to pool ──► [conn1]

─────────────────────────────────────────────────────────────────

POOL CONFIGURATION:

MaxIdleConnsPerHost: 10     ← how many idle connections per destination
MaxIdleConns: 100           ← total idle connections across all hosts
IdleConnTimeout: 90s        ← evict idle connections after this time

PITFALLS:
  ✗ new http.Client per request  → new pool each time → no reuse
  ✗ not closing resp.Body        → connection leaked, not returned to pool
  ✗ not draining body before close → server may be mid-send, conn discarded
```

---

## Diagram 4: JSON Marshal / Unmarshal Struct Tag Mapping

```
Go struct definition:

  type User struct {
      ID        int64     `json:"id"`
      FullName  string    `json:"name"`          ← renamed
      Email     string    `json:"email"`
      Password  string    `json:"-"`             ← always excluded
      Phone     string    `json:"phone,omitempty"` ← excluded if ""
      CreatedAt time.Time `json:"created_at"`
  }

  u := User{ID: 42, FullName: "Alice", Email: "alice@example.com",
            Password: "secret", CreatedAt: time.Now()}

json.Marshal(u) produces:
  {
      "id":         42,           ← Go field "ID" → JSON key "id"
      "name":       "Alice",      ← Go field "FullName" → JSON key "name"
      "email":      "alice@example.com",
                                  ← "Password" → excluded (json:"-")
                                  ← "Phone" → excluded (omitempty, value is "")
      "created_at": "2025-03-17T10:00:00Z"
  }

json.Unmarshal back:
  JSON key "id"         → Go field ID        (int64)
  JSON key "name"       → Go field FullName  (string)
  JSON key "email"      → Go field Email     (string)
  JSON key "created_at" → Go field CreatedAt (time.Time parsed from RFC3339)
  Extra JSON keys       → silently ignored by default

─────────────────────────────────────────────────────────────────

TAG SYNTAX: `json:"key,options"`

  Key options:
  "fieldName"          → use this JSON key name
  "-"                  → always omit (even if specified as "-,")
  ""                   → use Go field name as-is

  Behavior options:
  omitempty            → omit if zero value (0, false, "", nil, empty slice/map)
  string               → encode number as JSON string ("42" not 42)

  Examples:
  `json:"user_id"`           → key: "user_id"
  `json:"score,omitempty"`   → key: "score", omit if 0
  `json:"-"`                 → always excluded
  `json:",omitempty"`        → keep field name, omit if zero
  `json:"count,string"`      → encode int as "42" in JSON
```

---

## Diagram 5: Context Deadline Propagation Through HTTP Call Chain

```
INCOMING HTTP REQUEST from browser

  r.Context() — has a deadline from http.Server.WriteTimeout
       │
       ▼
  HTTP Handler
  ┌──────────────────────────────────────────────────────────┐
  │  ctx := r.Context()                                      │
  │  ctx has: deadline=T+10s, cancel when client disconnects │
  │                                                          │
  │  // Add tighter timeout for DB query                     │
  │  dbCtx, dbCancel := context.WithTimeout(ctx, 3s)         │
  │  defer dbCancel()                                        │
  │  dbCtx has: deadline=T+3s (inherits parent's cancel too) │
  │                                                          │
  │  user, err := db.QueryRowContext(dbCtx, ...)             │
  │       │                                                  │
  │       │ dbCtx.Deadline = T+3s                           │
  │       │                                                  │
  │  // Add timeout for downstream API call                  │
  │  apiCtx, apiCancel := context.WithTimeout(ctx, 5s)       │
  │  defer apiCancel()                                       │
  │  apiCtx has: deadline=T+5s                              │
  │                                                          │
  │  resp, err := httpClient.Do(req.WithContext(apiCtx))     │
  └──────────────────────────────────────────────────────────┘

CANCELLATION PROPAGATION (when client disconnects at T+2s):

  Browser disconnects
       │
       ▼
  r.Context() is CANCELLED (by net/http)
       │ cancellation propagates to ALL derived contexts
       ├──► dbCtx is CANCELLED → db.QueryRowContext returns ctx.Err()
       └──► apiCtx is CANCELLED → httpClient.Do returns ctx.Err()

DEADLINE HIERARCHY (child cannot exceed parent):

  context.Background() ─────────────────────────── no deadline
       │
  r.Context() ──────────────────────────────────── T + WriteTimeout
       │
  ├─► dbCtx (WithTimeout 3s) ────────────────────── T + min(3s, WriteTimeout)
  └─► apiCtx (WithTimeout 5s) ───────────────────── T + min(5s, WriteTimeout)

  Child deadline = min(child_timeout, parent_deadline)
```

---

## Diagram 6: Middleware Chain Wrapping

```
BUILDING THE CHAIN (setup time):

  mux := http.NewServeMux()
  mux.HandleFunc("/api", handler)

  // Chain: Recovery → Logger → Auth → mux
  h := Chain(mux, Recovery, Logger, Auth)

  // What happens inside Chain:
  // Start: h = mux
  // Apply Auth (last in list, applied first):   h = Auth(mux)
  // Apply Logger:                                h = Logger(Auth(mux))
  // Apply Recovery (first in list, outermost):  h = Recovery(Logger(Auth(mux)))

RUNTIME (when request arrives):

  Request
     │
     ▼
  Recovery.ServeHTTP(w, r)
     │  defer recover()
     │
     ▼
  Logger.ServeHTTP(w, r)
     │  start := time.Now()
     │
     ▼
  Auth.ServeHTTP(w, r)
     │  check Authorization header
     │  if invalid → return 401 (Logger and Recovery still run on return)
     │  if valid   ↓
     ▼
  mux.ServeHTTP(w, r)
     │  match route → call handler
     ▼
  handler(w, r)
     │  write response
     │
  ◄──── returns to Auth
  ◄──── returns to Logger (logs status code, duration)
  ◄──── returns to Recovery (no panic, nothing to recover)
  ◄──── returns to http.Server

─────────────────────────────────────────────────────────────────

PANIC SCENARIO:

  handler panics
     │
  ◄──── Auth returns (deferred functions ran)
  ◄──── Logger returns (deferred functions ran)
  ◄──── Recovery runs recover() ← panic intercepted here
         │ writes 500 to response
         │ logs panic with stack trace
  ◄──── returns to http.Server normally

KEY INSIGHT: Middleware wraps like Russian dolls.
  Outermost middleware (Recovery) is the first to start
  and the last to finish — it wraps everything inside it.
```
