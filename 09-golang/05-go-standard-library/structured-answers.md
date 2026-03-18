# Go Standard Library — Structured Answers

## Answer 1: How do you make an HTTP server in Go with proper timeouts?

A production HTTP server must set explicit timeouts on `http.Server` to prevent resource exhaustion from slow clients and runaway handlers.

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func newServer(addr string, handler http.Handler) *http.Server {
    return &http.Server{
        Addr:    addr,
        Handler: handler,

        // Time to read request headers + body from client
        ReadTimeout: 5 * time.Second,

        // Time to read only request headers (protects against Slowloris)
        ReadHeaderTimeout: 2 * time.Second,

        // Time to write entire response (starts after reading request)
        WriteTimeout: 10 * time.Second,

        // How long idle keep-alive connections are kept open
        IdleTimeout: 120 * time.Second,

        // Max bytes of request headers
        MaxHeaderBytes: 1 << 20, // 1MB
    }
}

func run() error {
    mux := http.NewServeMux()
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.Write([]byte(`{"status":"ok"}`))
    })

    srv := newServer(":8080", mux)

    // Start in background
    errCh := make(chan error, 1)
    go func() {
        log.Println("listening on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            errCh <- err
        }
        close(errCh)
    }()

    // Wait for signal or server error
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    select {
    case err := <-errCh:
        return err
    case sig := <-quit:
        log.Printf("received signal %v, shutting down", sig)
    }

    // Graceful shutdown: wait up to 30s for in-flight requests
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    return srv.Shutdown(ctx)
}

func main() {
    if err := run(); err != nil {
        log.Fatal(err)
    }
}
```

**Key points:**
- `ReadHeaderTimeout` specifically defends against Slowloris attacks
- `WriteTimeout` prevents handlers from holding connections open indefinitely
- `http.ErrServerClosed` is normal — returned when `Shutdown()` is called
- `Shutdown()` waits for in-flight requests; `Close()` forcibly closes everything

---

## Answer 2: How does `encoding/json` work? How do you customize marshaling?

`encoding/json` uses reflection to traverse struct fields and their tags at runtime.

**Standard operation:**

```go
type Person struct {
    Name      string     `json:"name"`
    Age       int        `json:"age"`
    Password  string     `json:"-"`              // never in JSON
    Phone     string     `json:"phone,omitempty"` // omit if empty string
    UpdatedAt *time.Time `json:"updated_at,omitempty"` // omit if nil
}

p := Person{Name: "Alice", Age: 30, Password: "secret"}
data, _ := json.Marshal(p)
// {"name":"Alice","age":30}  — Password excluded, Phone and UpdatedAt omitted

var decoded Person
json.Unmarshal(data, &decoded)
```

**Custom marshaling with `json.Marshaler` / `json.Unmarshaler`:**

```go
// Use case: store money as integer cents, serialize as decimal string
type Money struct {
    Cents int64 // 100 = $1.00
}

func (m Money) MarshalJSON() ([]byte, error) {
    dollars := float64(m.Cents) / 100
    return json.Marshal(fmt.Sprintf("%.2f", dollars))
}

func (m *Money) UnmarshalJSON(data []byte) error {
    var s string
    if err := json.Unmarshal(data, &s); err != nil {
        return err
    }
    f, err := strconv.ParseFloat(s, 64)
    if err != nil {
        return fmt.Errorf("parse money %q: %w", s, err)
    }
    m.Cents = int64(math.Round(f * 100))
    return nil
}

type Order struct {
    ID    int   `json:"id"`
    Total Money `json:"total"`
}

o := Order{ID: 1, Total: Money{Cents: 1999}}
data, _ := json.Marshal(o)
// {"id":1,"total":"19.99"}
```

**Streaming for large data:**

```go
// Writing large arrays without building them in memory
enc := json.NewEncoder(w)
for _, item := range hugeSlice {
    if err := enc.Encode(item); err != nil {
        return err
    }
}

// Reading large arrays without loading into memory
dec := json.NewDecoder(r)
dec.Token() // consume '['
for dec.More() {
    var item Item
    dec.Decode(&item)
    process(item)
}
dec.Token() // consume ']'
```

---

## Answer 3: How do you implement HTTP middleware in Go?

Go middleware follows a simple pattern: a function that takes an `http.Handler` and returns an `http.Handler`.

```go
// Middleware type signature
type Middleware func(http.Handler) http.Handler

// Example: logging middleware
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &wrappedWriter{ResponseWriter: w, statusCode: 200}

        next.ServeHTTP(rw, r)

        log.Printf("%s %s %d %s", r.Method, r.URL.Path, rw.statusCode, time.Since(start))
    })
}

// Example: auth middleware
func RequireAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return // short-circuit — next.ServeHTTP NOT called
        }
        claims, err := validateToken(token)
        if err != nil {
            http.Error(w, "invalid token", http.StatusUnauthorized)
            return
        }
        // Inject claims into context for handlers
        ctx := context.WithValue(r.Context(), claimsKey, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Chaining middlewares (outermost runs first)
func Chain(h http.Handler, mws ...Middleware) http.Handler {
    // Apply in reverse so first middleware in list is outermost
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}

// Usage:
mux := http.NewServeMux()
mux.HandleFunc("/api/data", dataHandler)

handler := Chain(mux,
    RecoveryMiddleware,  // outermost — handles panics from all others
    Logger,              // logs all requests
    RequireAuth,         // auth check
    RateLimiter(100),    // rate limiting
)
http.ListenAndServe(":8080", handler)
```

The `wrappedWriter` captures the status code written by the handler:

```go
type wrappedWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *wrappedWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

---

## Answer 4: How do you use context with HTTP requests for cancellation/timeout?

Context flows through the call chain and propagates cancellation automatically.

```go
// Server side: use request context for DB queries
func getUserHandler(w http.ResponseWriter, r *http.Request) {
    // r.Context() is cancelled when:
    // 1. Client disconnects
    // 2. Server's WriteTimeout fires
    // 3. Handler explicitly cancels (via WithCancel/WithTimeout on r.Context())
    ctx := r.Context()
    id := parseID(r)

    // Pass ctx to every downstream call
    user, err := userStore.GetUser(ctx, id)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            // Client disconnected — no need to log as error
            return
        }
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "request timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    json.NewEncoder(w).Encode(user)
}

// Client side: attach context to outgoing requests
func callDownstream(ctx context.Context, url string) (*Response, error) {
    // Add a 5-second timeout to the outgoing request
    reqCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(reqCtx, http.MethodGet, url, nil)
    if err != nil {
        return nil, fmt.Errorf("create request: %w", err)
    }

    resp, err := httpClient.Do(req)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return nil, fmt.Errorf("downstream call timed out: %w", err)
        }
        return nil, fmt.Errorf("downstream call failed: %w", err)
    }
    defer resp.Body.Close()
    // ...
}
```

---

## Answer 5: How do you read and write files efficiently in Go?

**Reading:**

```go
// Small files: read entirely into memory
data, err := os.ReadFile("config.json")

// Large files: stream with bufio.Scanner (line-by-line)
f, err := os.Open("large.log")
defer f.Close()
scanner := bufio.NewScanner(f)
for scanner.Scan() {
    process(scanner.Text())
}

// Large files: stream with bufio.Reader (arbitrary reads)
reader := bufio.NewReaderSize(f, 64*1024) // 64KB buffer
buf := make([]byte, 4096)
for {
    n, err := reader.Read(buf)
    if n > 0 {
        process(buf[:n])
    }
    if err == io.EOF {
        break
    }
    if err != nil {
        return err
    }
}
```

**Writing:**

```go
// Small files: write entirely
err := os.WriteFile("output.json", data, 0644)

// Large files: stream with bufio.Writer
f, err := os.Create("output.txt")
defer f.Close()
w := bufio.NewWriterSize(f, 64*1024) // 64KB buffer
defer w.Flush() // flush remaining buffer on function exit

for _, line := range lines {
    fmt.Fprintln(w, line) // writes to buffer, not file
}
// Flush writes buffer to file

// Atomic write: use temp file + rename
func writeAtomic(path string, data []byte) error {
    dir := filepath.Dir(path)
    tmp, err := os.CreateTemp(dir, ".tmp-*")
    if err != nil {
        return err
    }
    tmpPath := tmp.Name()
    if _, err := tmp.Write(data); err != nil {
        tmp.Close()
        os.Remove(tmpPath)
        return err
    }
    if err := tmp.Sync(); err != nil { // fsync before rename
        tmp.Close()
        os.Remove(tmpPath)
        return err
    }
    tmp.Close()
    return os.Rename(tmpPath, path) // atomic on same filesystem
}
```

---

## Answer 6: What is `io.Reader` and why is it so composable?

`io.Reader` is the cornerstone of Go's I/O model. Its single method — `Read(p []byte) (n int, err error)` — is simple enough that any byte source can implement it, yet powerful enough to model any sequential byte stream.

**Why it is composable:**

```go
// Any of these can be passed where io.Reader is expected:
os.File                    // disk files
*bytes.Buffer              // in-memory buffer
*strings.Reader            // string as reader
net.Conn                   // network connection
*gzip.Reader               // decompresses as you read
*cipher.StreamReader       // decrypts as you read
*bufio.Reader              // buffers reads for efficiency
http.Response.Body         // HTTP response body
io.LimitReader(r, n)       // stops after n bytes
io.MultiReader(r1, r2...)  // concatenates readers

// Example: chain readers to decompress + decrypt + hash + read
r := resp.Body
r = io.LimitReader(r, 10*1024*1024) // max 10MB
gr, _ := gzip.NewReader(r)          // decompress
defer gr.Close()
h := sha256.New()
tee := io.TeeReader(gr, h)          // hash while reading
br := bufio.NewReader(tee)          // buffer for efficiency

scanner := bufio.NewScanner(br)
for scanner.Scan() {
    process(scanner.Text())
}

checksum := fmt.Sprintf("%x", h.Sum(nil))
```

**The contract of `Read`:**
- Returns up to `len(p)` bytes
- Returns `io.EOF` (possibly with data) when the stream ends
- May return `n > 0` AND a non-nil error simultaneously
- A return of `n == 0, err == nil` means "try again" (valid but unusual)

---

## Answer 7: How do you make concurrent HTTP requests with error handling?

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

type Result struct {
    URL    string
    Data   []byte
    Status int
    Err    error
}

// FetchConcurrent fetches all URLs concurrently, respecting context cancellation.
func FetchConcurrent(ctx context.Context, urls []string) ([]Result, error) {
    client := &http.Client{Timeout: 15 * time.Second}
    results := make([]Result, len(urls))
    var wg sync.WaitGroup

    for i, url := range urls {
        wg.Add(1)
        go func(idx int, u string) {
            defer wg.Done()
            results[idx] = doFetch(ctx, client, u)
        }(i, url)
    }
    wg.Wait()

    // Collect errors
    var errs []error
    for _, r := range results {
        if r.Err != nil {
            errs = append(errs, fmt.Errorf("%s: %w", r.URL, r.Err))
        }
    }
    return results, errors.Join(errs...)
}

func doFetch(ctx context.Context, c *http.Client, url string) Result {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return Result{URL: url, Err: err}
    }

    resp, err := c.Do(req)
    if err != nil {
        return Result{URL: url, Err: err}
    }
    defer resp.Body.Close()

    // Drain and close for connection reuse
    body, err := io.ReadAll(io.LimitReader(resp.Body, 5<<20)) // 5MB max
    if err != nil {
        return Result{URL: url, Status: resp.StatusCode, Err: fmt.Errorf("read body: %w", err)}
    }

    if resp.StatusCode >= 400 {
        return Result{
            URL:    url,
            Status: resp.StatusCode,
            Err:    fmt.Errorf("HTTP error %d", resp.StatusCode),
        }
    }

    return Result{URL: url, Status: resp.StatusCode, Data: body}
}
```

---

## Answer 8: How does structured logging work with `log/slog` (Go 1.21+)?

`log/slog` provides a structured, leveled logging API. Log entries are key-value pairs rather than formatted strings, making them machine-parseable.

```go
package main

import (
    "context"
    "log/slog"
    "os"
)

func main() {
    // Text handler (human-readable)
    textLogger := slog.New(slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
    }))

    // JSON handler (machine-readable for production/log aggregators)
    jsonLogger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level:     slog.LevelInfo,
        AddSource: true, // include file:line
    }))
    slog.SetDefault(jsonLogger)

    // Basic usage — alternating key/value pairs
    slog.Info("server started", "addr", ":8080", "pid", os.Getpid())
    // {"time":"2025-03-17T10:00:00Z","level":"INFO","msg":"server started","addr":":8080","pid":12345}

    // With structured attributes (type-safe)
    slog.Info("request received",
        slog.String("method", "GET"),
        slog.String("path", "/api/users"),
        slog.Int("status", 200),
        slog.Duration("latency", 5*time.Millisecond),
    )

    // Child logger with pre-set fields
    requestLogger := slog.Default().With(
        "request_id", "abc-123",
        "user_id", 42,
    )
    requestLogger.Info("processing")
    requestLogger.Error("failed", "err", err)

    // Groups — namespace related fields
    slog.Info("database query",
        slog.Group("db",
            slog.String("host", "localhost"),
            slog.Int("port", 5432),
            slog.Duration("query_time", 2*time.Millisecond),
        ),
    )
    // {"msg":"database query","db":{"host":"localhost","port":5432,"query_time":"2ms"}}

    // Context-aware logging
    ctx := context.WithValue(context.Background(), requestIDKey, "req-001")
    slog.InfoContext(ctx, "context-aware log")

    // Levels
    slog.Debug("verbose debug info")   // filtered if level > Debug
    slog.Info("normal operational")
    slog.Warn("something unexpected")
    slog.Error("something failed", "err", err)
}
```

Custom handler example:

```go
// Add request ID from context to all log entries
type ContextHandler struct {
    slog.Handler
}

func (h ContextHandler) Handle(ctx context.Context, r slog.Record) error {
    if id, ok := ctx.Value(requestIDKey).(string); ok {
        r.AddAttrs(slog.String("request_id", id))
    }
    return h.Handler.Handle(ctx, r)
}
```

---

## Answer 9: How do you handle JSON streaming for large files?

For large JSON files, use `json.NewDecoder` with `Decode()` or `Token()` in a loop to process one item at a time:

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "os"
)

type LogEntry struct {
    Timestamp string `json:"ts"`
    Level     string `json:"level"`
    Message   string `json:"msg"`
    Fields    map[string]interface{} `json:"fields"`
}

// StreamJSONArray processes a JSON array without loading it all into memory.
func StreamJSONArray(r io.Reader, process func(LogEntry) error) error {
    dec := json.NewDecoder(r)

    // Expect array open '['
    t, err := dec.Token()
    if err != nil {
        return fmt.Errorf("read array open: %w", err)
    }
    if delim, ok := t.(json.Delim); !ok || delim != '[' {
        return fmt.Errorf("expected '[', got %v", t)
    }

    var count int
    for dec.More() {
        var entry LogEntry
        if err := dec.Decode(&entry); err != nil {
            return fmt.Errorf("decode entry %d: %w", count, err)
        }
        if err := process(entry); err != nil {
            return fmt.Errorf("process entry %d: %w", count, err)
        }
        count++
    }

    // Consume array close ']'
    if _, err := dec.Token(); err != nil {
        return fmt.Errorf("read array close: %w", err)
    }

    fmt.Printf("processed %d entries\n", count)
    return nil
}

// StreamJSONLines handles newline-delimited JSON (one JSON object per line)
func StreamJSONLines(r io.Reader, process func(LogEntry) error) error {
    dec := json.NewDecoder(r)
    var count int
    for dec.More() {
        var entry LogEntry
        if err := dec.Decode(&entry); err != nil {
            return fmt.Errorf("decode line %d: %w", count, err)
        }
        if err := process(entry); err != nil {
            return fmt.Errorf("process line %d: %w", count, err)
        }
        count++
    }
    return nil
}

func main() {
    f, err := os.Open("logs.json")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    err = StreamJSONArray(f, func(e LogEntry) error {
        if e.Level == "ERROR" {
            fmt.Printf("[%s] %s\n", e.Timestamp, e.Message)
        }
        return nil
    })
    if err != nil {
        fmt.Fprintln(os.Stderr, "stream error:", err)
        os.Exit(1)
    }
}
```

---

## Answer 10: How do you configure a production-ready `http.Client`?

```go
package httpclient

import (
    "crypto/tls"
    "net"
    "net/http"
    "time"
)

// NewProductionClient creates an http.Client suitable for production use.
func NewProductionClient() *http.Client {
    dialer := &net.Dialer{
        Timeout:   5 * time.Second,  // TCP connection establishment timeout
        KeepAlive: 30 * time.Second, // TCP keep-alive period
        DualStack: true,             // use both IPv4 and IPv6
    }

    transport := &http.Transport{
        DialContext: dialer.DialContext,

        // TLS
        TLSHandshakeTimeout: 10 * time.Second,
        TLSClientConfig: &tls.Config{
            MinVersion: tls.VersionTLS12,
        },

        // Connection pool
        MaxIdleConns:          100,              // total idle connections across all hosts
        MaxIdleConnsPerHost:   10,               // idle connections per host (default: 2)
        MaxConnsPerHost:       0,                // 0 = unlimited (default)
        IdleConnTimeout:       90 * time.Second, // idle connection eviction time

        // Protocol
        DisableCompression:    false, // allow Accept-Encoding: gzip
        ForceAttemptHTTP2:     true,  // try HTTP/2 if available
        ExpectContinueTimeout: 1 * time.Second,

        // Response
        ResponseHeaderTimeout: 10 * time.Second, // time to wait for response headers
    }

    return &http.Client{
        Timeout:   30 * time.Second, // end-to-end timeout for the entire request
        Transport: transport,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            if len(via) >= 10 {
                return fmt.Errorf("too many redirects (%d)", len(via))
            }
            // Preserve Authorization header on redirect to same host only
            if req.URL.Host != via[0].URL.Host {
                req.Header.Del("Authorization")
            }
            return nil
        },
    }
}

// Critical: the Transport must be shared and reused for connection pooling.
// Never create a new http.Client per request — the pool won't be used.
var DefaultClient = NewProductionClient()
```

**Common mistakes to avoid:**
1. Creating a new `http.Client` per request — defeats connection pooling
2. Using `http.DefaultClient` — no timeout
3. Not closing `resp.Body` — connection leak
4. Not draining `resp.Body` before close — connection not reused
5. Setting `Timeout` only on `Transport` fields but not on `Client.Timeout` — partial timeout coverage
