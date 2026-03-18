# Go Web & Microservices — Follow-Up Traps

---

## Trap 1: http.ListenAndServe Blocks — You Need a Goroutine for Graceful Shutdown

**The trap:** "I call `srv.Shutdown()` after `srv.ListenAndServe()` but it never runs."

**The problem:** `ListenAndServe` blocks indefinitely. Any code after it is unreachable until the server stops. You must run it in a goroutine.

```go
// WRONG: Shutdown() is unreachable
http.ListenAndServe(":8080", handler)  // blocks forever
srv.Shutdown(ctx)                        // NEVER reached

// CORRECT:
go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("server error: %v", err)
    }
}()

// Block here waiting for OS signal
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

Also note: `http.ErrServerClosed` is the expected error after `Shutdown()` is called — you must check for it explicitly or your program will fatally exit.

---

## Trap 2: ServeMux Routing — Trailing Slash Changes Behavior

**The trap:** "I registered `/api` but requests to `/api/users` return 404."

**The problem:** In `net/http.ServeMux`:
- `/api` matches **only** the exact path `/api`
- `/api/` (trailing slash) matches `/api/` **and all sub-paths** like `/api/users`, `/api/v1/foo`

```go
mux := http.NewServeMux()
mux.HandleFunc("/api", handler)   // matches ONLY /api
mux.HandleFunc("/api/", handler)  // matches /api/ AND /api/users AND /api/anything/else

// Also: the mux automatically redirects /api to /api/ if you registered /api/ only
// This can cause unexpected 301 redirects in production.
```

Go 1.22+ method routing also has implications:
```go
// These are different routes:
mux.HandleFunc("GET /users", listUsers)       // must be GET /users (no trailing slash)
mux.HandleFunc("GET /users/", listUsersTree)  // matches GET /users/anything
```

---

## Trap 3: Reading Request Body Twice Silently Returns Empty

**The trap:** "My logging middleware reads the body, and now my handler gets empty JSON."

**The problem:** `http.Request.Body` is an `io.ReadCloser` — it is a stream. Once read, the position is at EOF. The handler reads nothing.

```go
// Middleware reads body: handler gets nothing
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        body, _ := io.ReadAll(r.Body)
        log.Println(string(body))
        // r.Body is now at EOF! Handler will decode empty body.
        next.ServeHTTP(w, r)
    })
}

// Fix: restore the body
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        body, _ := io.ReadAll(r.Body)
        r.Body.Close()
        log.Println(string(body))

        // Restore for handler
        r.Body = io.NopCloser(bytes.NewReader(body))
        next.ServeHTTP(w, r)
    })
}
```

The silent failure is dangerous: `json.NewDecoder(r.Body).Decode(&req)` succeeds with no error but leaves req at zero values if the body is empty.

---

## Trap 4: Goroutine Leak from Unread Response Bodies in HTTP Clients

**The trap:** "My service gradually runs out of file descriptors/connections even though I'm closing responses."

**The problem:** `http.Response.Body` must be **both read to EOF AND closed** for the underlying TCP connection to be returned to the pool. Just calling `resp.Body.Close()` without draining is not sufficient when the body isn't fully read.

```go
// LEAK: closes without draining — connection cannot be reused
resp, err := http.Get(url)
if err != nil {
    return err
}
resp.Body.Close()  // connection goes to TIME_WAIT, not reused

// CORRECT: drain before closing
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
io.Copy(io.Discard, resp.Body)  // drain remaining body

// Also correct: fully read the body
body, err := io.ReadAll(resp.Body)
resp.Body.Close()
```

Always use `defer resp.Body.Close()` AND read the body (or drain it):
```go
resp, err := client.Do(req)
if err != nil {
    return nil, err
}
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()
```

---

## Trap 5: Panic in Handler Without Recovery Middleware Crashes the Server

**The trap:** "A nil pointer dereference in one handler crashes my entire server."

**The problem:** Go's `net/http` server spawns a goroutine per connection. If a handler panics, that goroutine crashes — but `net/http` actually does recover from handler panics internally (since Go 1.0) and logs to stderr. **However**: it returns a 500 with no body, closes the connection, and if you have a custom logger or need to return structured error JSON, you won't get it.

```go
// http.Server recovers panics but gives you no control:
// - Writes nothing to the response body
// - Logs to stderr with default format
// - The client sees connection closed / empty 500

// Correct: add your own recovery middleware for:
// 1. Structured error response JSON
// 2. Custom logging with stack trace and request context
// 3. Alerting / error tracking (Sentry, etc.)

func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if p := recover(); p != nil {
                log.Printf("PANIC: %v\n%s", p, debug.Stack())
                w.Header().Set("Content-Type", "application/json")
                w.WriteHeader(http.StatusInternalServerError)
                json.NewEncoder(w).Encode(map[string]string{
                    "error":      "internal server error",
                    "request_id": getRequestID(r.Context()),
                })
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

---

## Trap 6: json.Decoder vs json.Unmarshal — Which Is Better for HTTP?

**The trap:** "I use `json.Unmarshal(body, &req)` in all my handlers. Is that fine?"

**The nuance:**

```go
// json.Unmarshal: requires reading entire body into memory first
body, err := io.ReadAll(r.Body)
if err != nil { /* ... */ }
var req MyRequest
json.Unmarshal(body, &req)  // processes the whole []byte at once

// json.NewDecoder: streams from the reader — no need to buffer entire body
json.NewDecoder(r.Body).Decode(&req)  // streams directly from r.Body
```

**Use `json.NewDecoder` for HTTP handlers** because:
1. Body may be large — no need to buffer it all
2. No intermediate `[]byte` allocation
3. More idiomatic for streaming input

**Exception — use `json.Unmarshal` when:**
1. You already have the data as `[]byte` (e.g., from a cache or Kafka)
2. You need to decode multiple values from one buffer
3. You are NOT in an HTTP context

**One important difference:**

```go
// json.Decoder allows multiple JSON values in sequence:
dec := json.NewDecoder(r.Body)
for dec.More() {
    var item Item
    dec.Decode(&item)
}

// json.Unmarshal fails if there is trailing data after the JSON
json.Unmarshal([]byte(`{"a":1}  extra`), &v)  // error: invalid character 'e'
```

---

## Trap 7: Using context.Background() Instead of r.Context()

**The trap:** "I pass `context.Background()` to my DB query. What's wrong with that?"

**The problem:** You lose all of these benefits that `r.Context()` provides:

```go
// r.Context() is cancelled when:
// 1. Client disconnects (TCP RST)
// 2. Server ReadTimeout expires
// 3. You cancel it yourself

// With context.Background(): DB query continues even after client left
func badHandler(w http.ResponseWriter, r *http.Request) {
    rows, err := db.QueryContext(context.Background(), // BUG
        "SELECT * FROM large_table WHERE ...")
    // If client disconnects, this query keeps running until DB timeout (could be minutes)
}

// Correct: cancel propagates to DB automatically
func goodHandler(w http.ResponseWriter, r *http.Request) {
    rows, err := db.QueryContext(r.Context(), // propagates client disconnect
        "SELECT * FROM large_table WHERE ...")
    if err != nil {
        if errors.Is(err, context.Canceled) {
            return // client disconnected — don't bother responding
        }
    }
}
```

Also: request context carries trace IDs, auth claims, and any values set by middleware. Using `context.Background()` loses all of that.

---

## Trap 8: CORS Headers Must Be Set Before Writing Status Code

**The trap:** "My CORS middleware sets headers but the browser still blocks the request."

**The problem:** In HTTP, headers must be written before the body and status code. Once `w.WriteHeader(status)` is called (or the first `w.Write`), headers are flushed and cannot be added.

```go
// BROKEN: headers set after WriteHeader
func handler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)  // headers flushed here!
    w.Header().Set("Access-Control-Allow-Origin", "*")  // too late — ignored
    json.NewEncoder(w).Encode(data)
}

// BROKEN MIDDLEWARE: handler called first
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        next.ServeHTTP(w, r)  // handler runs, may write headers
        w.Header().Set("Access-Control-Allow-Origin", "*")  // too late!
    })
}

// CORRECT: set headers before calling next
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")  // set first
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

---

## Trap 9: Large File Upload — Don't Buffer Entire Body in Memory

**The trap:** "I use `io.ReadAll(r.Body)` for uploads. Should be fine for a 500MB video upload?"

**The problem:** `io.ReadAll` loads the entire content into a single `[]byte` in memory. For a 500MB upload on 100 concurrent requests: 50GB of memory just for uploads!

```go
// WRONG: loads entire file into memory
func uploadHandler(w http.ResponseWriter, r *http.Request) {
    data, err := io.ReadAll(r.Body)  // 500MB in RAM!
    os.WriteFile("upload.bin", data, 0644)
}

// CORRECT: stream directly to disk
func uploadHandler(w http.ResponseWriter, r *http.Request) {
    // Enforce size limit before reading
    r.Body = http.MaxBytesReader(w, r.Body, 500<<20) // 500MB limit

    dst, err := os.CreateTemp("", "upload-*")
    if err != nil {
        http.Error(w, "cannot create temp file", 500)
        return
    }
    defer dst.Close()

    // Streams body to disk — memory usage is only the copy buffer (~32KB)
    written, err := io.Copy(dst, r.Body)
    if err != nil {
        // Check specifically for MaxBytesReader limit exceeded
        var maxBytesErr *http.MaxBytesError
        if errors.As(err, &maxBytesErr) {
            http.Error(w, "file too large", http.StatusRequestEntityTooLarge)
            return
        }
        http.Error(w, "upload failed", 500)
        return
    }
    // written = bytes written to disk
    _ = written
}
```

For multipart forms, use `r.ParseMultipartForm(32 << 20)` which spills to disk after 32MB, preventing memory exhaustion.

---

## Trap 10: http.Client Without Timeout Hangs Forever

**The trap:** "The default `http.Client` or `http.Get` is fine for calling external APIs."

**The problem:** The default `http.Client` has **no timeout**. If the remote server hangs (e.g., accepts the connection but never responds), your goroutine hangs forever — causing a goroutine leak.

```go
// DANGEROUS: no timeout, can hang indefinitely
resp, err := http.Get("https://slow-api.example.com/data")

// Also dangerous: default client
resp, err := http.DefaultClient.Do(req)

// CORRECT: always set a timeout
client := &http.Client{
    Timeout: 10 * time.Second, // total request timeout (connect + headers + body)
    Transport: &http.Transport{
        DialContext:           (&net.Dialer{Timeout: 3 * time.Second}).DialContext,
        TLSHandshakeTimeout:   3 * time.Second,
        ResponseHeaderTimeout: 5 * time.Second,
        IdleConnTimeout:       90 * time.Second,
    },
}

// Even better: per-request timeout via context
ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
defer cancel()
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := client.Do(req)
```

---

## Trap 11: Middleware Order Matters — Inner vs Outer

**The trap:** "My auth middleware runs after my logging middleware logged the request, but authentication errors don't get logged properly."

**The problem:** Middleware wraps like Russian nesting dolls. The order you apply them determines execution order.

```go
// Applying: A(B(C(handler)))
// Execution order:
//   Request:  A → B → C → handler
//   Response: handler → C → B → A

// If Logging wraps Auth:
handler = LoggingMiddleware(AuthMiddleware(mux))
// Order: Logging → Auth → handler
// Logging runs BEFORE auth — it logs every request including rejected ones
// This is usually what you want (log auth failures)

// If Auth wraps Logging:
handler = AuthMiddleware(LoggingMiddleware(mux))
// Order: Auth → Logging → handler
// Unauthenticated requests are rejected BEFORE logging — they won't appear in logs
// This means you have blind spots in your audit log!

// Recommended order (outermost first):
handler = Chain(mux,
    RequestID,    // inject ID first (available to all subsequent middleware)
    Logging,      // log everything including auth failures
    Recovery,     // catch panics from auth and handler
    RateLimit,    // drop excessive requests before auth
    Auth,         // authenticate validated, rate-limited requests
    Compress,     // compress response after handler runs
)
```

---

## Trap 12: JSON Encoder Writes 200 Status Implicitly

**The trap:** "I want to return 201 Created but the client always sees 200."

**The problem:** The first call to `w.Write()` (including `json.NewEncoder(w).Encode()`) implicitly calls `w.WriteHeader(200)`. You must call `w.WriteHeader(status)` BEFORE writing the body.

```go
// WRONG: 200 is sent before WriteHeader(201) can take effect
func createHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(created)  // implicit WriteHeader(200) here!
    w.WriteHeader(http.StatusCreated)   // ignored — headers already sent
}

// CORRECT:
func createHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)   // must come BEFORE any Write
    json.NewEncoder(w).Encode(created)  // writes body with 201 status
}
```

The Go documentation states this clearly but it's a frequent mistake. You can verify with the `net/http/httptest.ResponseRecorder` in tests — `w.Code` will show the actual status.
