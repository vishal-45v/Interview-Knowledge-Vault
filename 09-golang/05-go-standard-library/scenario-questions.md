# Go Standard Library — Scenario Questions

## HTTP Server Scenarios

**Scenario 1: Write an HTTP server that handles `/health` and `/api/users` endpoints with proper timeouts.**

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "time"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

var users = []User{
    {1, "Alice", "alice@example.com"},
    {2, "Bob", "bob@example.com"},
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "ok", "time": time.Now().UTC().Format(time.RFC3339)})
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(users)
    case http.MethodPost:
        var u User
        if err := json.NewDecoder(r.Body).Decode(&u); err != nil {
            http.Error(w, "invalid JSON body", http.StatusBadRequest)
            return
        }
        u.ID = len(users) + 1
        users = append(users, u)
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusCreated)
        json.NewEncoder(w).Encode(u)
    default:
        http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
    }
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/health", healthHandler)
    mux.HandleFunc("/api/users", usersHandler)

    server := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadTimeout:       5 * time.Second,
        ReadHeaderTimeout: 2 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       120 * time.Second,
    }

    log.Println("server listening on :8080")
    log.Fatal(server.ListenAndServe())
}
```

Key points: custom `http.Server` with timeouts, method routing in handlers, `json.NewEncoder(w).Encode()` for streaming response, proper Content-Type header.

---

**Scenario 2: Implement an HTTP middleware for request logging.**

```go
package middleware

import (
    "log"
    "net/http"
    "time"
)

// responseWriter wraps http.ResponseWriter to capture the status code.
type responseWriter struct {
    http.ResponseWriter
    statusCode int
    bytes      int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    n, err := rw.ResponseWriter.Write(b)
    rw.bytes += n
    return n, err
}

// Logger returns a middleware that logs every request.
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(rw, r)

        log.Printf(
            "%s %s %d %d bytes %s",
            r.Method,
            r.URL.Path,
            rw.statusCode,
            rw.bytes,
            time.Since(start),
        )
    })
}

// Chaining multiple middlewares
func Chain(h http.Handler, middlewares ...func(http.Handler) http.Handler) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// Usage
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)

    handler := Chain(mux,
        Logger,
        AuthMiddleware,
        RecoveryMiddleware,
    )
    http.ListenAndServe(":8080", handler)
}
```

---

**Scenario 3: Implement request-scoped data passing through middleware using context.**

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "github.com/google/uuid"
)

type contextKey string

const (
    requestIDKey contextKey = "request_id"
    userIDKey    contextKey = "user_id"
)

// RequestID middleware injects a unique ID into the context.
func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" {
            id = uuid.NewString()
        }
        ctx := context.WithValue(r.Context(), requestIDKey, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Helper to extract request ID from context.
func GetRequestID(ctx context.Context) string {
    id, _ := ctx.Value(requestIDKey).(string)
    return id
}

// Handler reads from context.
func myHandler(w http.ResponseWriter, r *http.Request) {
    reqID := GetRequestID(r.Context())
    fmt.Fprintf(w, "request ID: %s\n", reqID)
}
```

---

## JSON and File Scenarios

**Scenario 4: Read a large JSON file without loading it all into memory. How?**

Use a streaming JSON decoder (`json.NewDecoder`) with `Token()` or `Decode()` in a loop:

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

// ProcessLargeJSONArray streams a JSON array without loading it all into memory.
// Expected format: [{"id":1,...}, {"id":2,...}, ...]
func ProcessLargeJSONArray(filename string, process func(Product) error) error {
    f, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("open %s: %w", filename, err)
    }
    defer f.Close()

    dec := json.NewDecoder(f)

    // Read opening bracket '['
    if _, err := dec.Token(); err != nil {
        return fmt.Errorf("read opening token: %w", err)
    }

    // Stream each element
    var count int
    for dec.More() {
        var p Product
        if err := dec.Decode(&p); err != nil {
            return fmt.Errorf("decode product at index %d: %w", count, err)
        }
        if err := process(p); err != nil {
            return fmt.Errorf("process product %d: %w", p.ID, err)
        }
        count++
    }

    // Read closing bracket ']'
    if _, err := dec.Token(); err != nil {
        return fmt.Errorf("read closing token: %w", err)
    }

    fmt.Printf("processed %d products\n", count)
    return nil
}

func main() {
    err := ProcessLargeJSONArray("products.json", func(p Product) error {
        fmt.Printf("product %d: %s $%.2f\n", p.ID, p.Name, p.Price)
        return nil
    })
    if err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

Memory usage: only one `Product` struct at a time, regardless of file size.

---

**Scenario 5: Read a large CSV file line by line and process each record.**

```go
package main

import (
    "bufio"
    "encoding/csv"
    "fmt"
    "os"
    "strconv"
)

type Transaction struct {
    ID     int
    Amount float64
    User   string
}

func processCSV(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("open %s: %w", filename, err)
    }
    defer f.Close()

    // bufio.NewReader improves performance with a buffer
    reader := csv.NewReader(bufio.NewReader(f))
    reader.FieldsPerRecord = 3 // enforce column count

    // Skip header
    if _, err := reader.Read(); err != nil {
        return fmt.Errorf("read header: %w", err)
    }

    var total float64
    lineNum := 1

    for {
        record, err := reader.Read()
        if err != nil {
            if err.Error() == "EOF" {
                break
            }
            return fmt.Errorf("read line %d: %w", lineNum, err)
        }
        lineNum++

        id, err := strconv.Atoi(record[0])
        if err != nil {
            return fmt.Errorf("line %d: invalid id %q: %w", lineNum, record[0], err)
        }
        amount, err := strconv.ParseFloat(record[1], 64)
        if err != nil {
            return fmt.Errorf("line %d: invalid amount %q: %w", lineNum, record[1], err)
        }

        t := Transaction{ID: id, Amount: amount, User: record[2]}
        total += t.Amount
        fmt.Printf("tx %d: $%.2f by %s\n", t.ID, t.Amount, t.User)
    }

    fmt.Printf("total: $%.2f across %d transactions\n", total, lineNum-1)
    return nil
}
```

---

## Parallel HTTP Requests

**Scenario 6: Make parallel HTTP requests with a timeout and aggregate results.**

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

type FetchResult struct {
    URL    string
    Body   string
    Status int
    Err    error
}

func FetchAll(ctx context.Context, urls []string) []FetchResult {
    client := &http.Client{Timeout: 10 * time.Second}
    results := make([]FetchResult, len(urls))
    var wg sync.WaitGroup

    for i, url := range urls {
        wg.Add(1)
        go func(idx int, u string) {
            defer wg.Done()
            results[idx] = fetch(ctx, client, u)
        }(i, url)
    }

    wg.Wait()
    return results
}

func fetch(ctx context.Context, client *http.Client, url string) FetchResult {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return FetchResult{URL: url, Err: fmt.Errorf("create request: %w", err)}
    }

    resp, err := client.Do(req)
    if err != nil {
        return FetchResult{URL: url, Err: fmt.Errorf("do request: %w", err)}
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(io.LimitReader(resp.Body, 1<<20)) // max 1MB
    if err != nil {
        return FetchResult{URL: url, Status: resp.StatusCode, Err: fmt.Errorf("read body: %w", err)}
    }

    return FetchResult{URL: url, Status: resp.StatusCode, Body: string(body)}
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()

    urls := []string{
        "https://httpbin.org/get",
        "https://httpbin.org/delay/1",
        "https://httpbin.org/status/404",
    }

    results := FetchAll(ctx, urls)
    for _, r := range results {
        if r.Err != nil {
            fmt.Printf("FAIL %s: %v\n", r.URL, r.Err)
        } else {
            fmt.Printf("OK   %s: %d (%d bytes)\n", r.URL, r.Status, len(r.Body))
        }
    }
}
```

---

**Scenario 7: Use a semaphore to limit concurrent HTTP requests to 5 at a time.**

```go
func FetchWithConcurrencyLimit(ctx context.Context, urls []string, maxConcurrent int) []FetchResult {
    client := &http.Client{Timeout: 10 * time.Second}
    results := make([]FetchResult, len(urls))
    semaphore := make(chan struct{}, maxConcurrent) // buffered channel as semaphore
    var wg sync.WaitGroup

    for i, url := range urls {
        wg.Add(1)
        go func(idx int, u string) {
            defer wg.Done()
            semaphore <- struct{}{}         // acquire slot
            defer func() { <-semaphore }() // release slot
            results[idx] = fetch(ctx, client, u)
        }(i, url)
    }

    wg.Wait()
    return results
}
```

---

## File I/O Scenarios

**Scenario 8: Efficiently read a file and count word frequencies.**

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "sort"
    "strings"
    "unicode"
)

type WordCount struct {
    Word  string
    Count int
}

func WordFrequency(filename string) ([]WordCount, error) {
    f, err := os.Open(filename)
    if err != nil {
        return nil, fmt.Errorf("open %s: %w", filename, err)
    }
    defer f.Close()

    freq := make(map[string]int)
    scanner := bufio.NewScanner(f)

    // Use custom split to tokenize words
    scanner.Split(func(data []byte, atEOF bool) (advance int, token []byte, err error) {
        // Skip non-letter characters
        start := 0
        for start < len(data) && !unicode.IsLetter(rune(data[start])) {
            start++
        }
        for end := start; end < len(data); end++ {
            if !unicode.IsLetter(rune(data[end])) {
                return end + 1, data[start:end], nil
            }
        }
        if atEOF && start < len(data) {
            return len(data), data[start:], nil
        }
        return start, nil, nil
    })

    for scanner.Scan() {
        word := strings.ToLower(scanner.Text())
        if word != "" {
            freq[word]++
        }
    }
    if err := scanner.Err(); err != nil {
        return nil, fmt.Errorf("scan %s: %w", filename, err)
    }

    counts := make([]WordCount, 0, len(freq))
    for w, c := range freq {
        counts = append(counts, WordCount{w, c})
    }
    sort.Slice(counts, func(i, j int) bool {
        return counts[i].Count > counts[j].Count
    })
    return counts, nil
}
```

---

**Scenario 9: Write a function that copies a file safely (atomic write using temp file + rename).**

```go
package main

import (
    "fmt"
    "io"
    "os"
    "path/filepath"
)

// SafeCopyFile copies src to dst atomically using a temp file + rename.
// Avoids partial writes being visible to other processes.
func SafeCopyFile(dst, src string) error {
    srcFile, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("open source %s: %w", src, err)
    }
    defer srcFile.Close()

    srcInfo, err := srcFile.Stat()
    if err != nil {
        return fmt.Errorf("stat source: %w", err)
    }

    // Write to a temp file in the same directory as dst (same filesystem)
    dir := filepath.Dir(dst)
    tmpFile, err := os.CreateTemp(dir, ".safecopy-*")
    if err != nil {
        return fmt.Errorf("create temp file: %w", err)
    }
    tmpPath := tmpFile.Name()

    // Ensure cleanup on failure
    success := false
    defer func() {
        tmpFile.Close()
        if !success {
            os.Remove(tmpPath) // best-effort cleanup
        }
    }()

    // Copy content
    if _, err := io.Copy(tmpFile, srcFile); err != nil {
        return fmt.Errorf("copy content: %w", err)
    }

    // Sync to disk before rename
    if err := tmpFile.Sync(); err != nil {
        return fmt.Errorf("sync temp file: %w", err)
    }
    tmpFile.Close()

    // Set permissions to match source
    if err := os.Chmod(tmpPath, srcInfo.Mode()); err != nil {
        return fmt.Errorf("chmod temp file: %w", err)
    }

    // Atomic rename — either the old file or new file is visible, never partial
    if err := os.Rename(tmpPath, dst); err != nil {
        return fmt.Errorf("rename to %s: %w", dst, err)
    }

    success = true
    return nil
}
```

---

**Scenario 10: Implement a rate-limited HTTP handler using the standard library only.**

```go
package main

import (
    "net/http"
    "sync"
    "time"
)

// TokenBucket implements a simple rate limiter.
type TokenBucket struct {
    mu       sync.Mutex
    tokens   float64
    capacity float64
    rate     float64 // tokens per second
    lastTime time.Time
}

func NewTokenBucket(rate, capacity float64) *TokenBucket {
    return &TokenBucket{
        tokens:   capacity,
        capacity: capacity,
        rate:     rate,
        lastTime: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(tb.lastTime).Seconds()
    tb.lastTime = now

    tb.tokens += elapsed * tb.rate
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }

    if tb.tokens >= 1.0 {
        tb.tokens -= 1.0
        return true
    }
    return false
}

// RateLimit wraps a handler with rate limiting.
func RateLimit(limiter *TokenBucket, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow() {
            w.Header().Set("Retry-After", "1")
            http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

---

**Scenario 11: Implement HTTP server graceful shutdown.**

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

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(2 * time.Second) // simulate slow handler
        w.Write([]byte("hello"))
    })

    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }

    // Start server in background goroutine
    go func() {
        log.Println("listening on :8080")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("ListenAndServe: %v", err)
        }
    }()

    // Wait for OS signal (Ctrl+C or kill)
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("shutting down server...")

    // Give in-flight requests up to 30 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Printf("server forced to shutdown: %v", err)
    }

    log.Println("server stopped")
}
```

---

**Scenario 12: Implement an HTTP client with custom Transport for mocking in tests.**

```go
package httpclient

import (
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

// Client wraps http.Client with domain logic.
type Client struct {
    httpClient *http.Client
    baseURL    string
}

// New creates a production client.
func New(baseURL string) *Client {
    return &Client{
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
            Transport: &http.Transport{
                MaxIdleConnsPerHost: 10,
                IdleConnTimeout:     90 * time.Second,
            },
        },
        baseURL: baseURL,
    }
}

// NewWithTransport creates a client with a custom transport (for testing).
func NewWithTransport(baseURL string, transport http.RoundTripper) *Client {
    return &Client{
        httpClient: &http.Client{Transport: transport},
        baseURL:    baseURL,
    }
}

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func (c *Client) GetUser(id int) (*User, error) {
    resp, err := c.httpClient.Get(fmt.Sprintf("%s/users/%d", c.baseURL, id))
    if err != nil {
        return nil, fmt.Errorf("get user %d: %w", id, err)
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusNotFound {
        return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
    }
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("get user %d: unexpected status %d", id, resp.StatusCode)
    }

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, fmt.Errorf("decode user response: %w", err)
    }
    return &user, nil
}
```

---

**Scenario 13: How do you implement a timeout-aware database query using context?**

```go
package store

import (
    "context"
    "database/sql"
    "fmt"
    "time"
)

type UserStore struct {
    db *sql.DB
}

func (s *UserStore) GetUser(ctx context.Context, id int64) (*User, error) {
    // Add timeout if not already present in context
    queryCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    var u User
    err := s.db.QueryRowContext(queryCtx,
        "SELECT id, name, email, created_at FROM users WHERE id = $1", id,
    ).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)

    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
        }
        if ctx.Err() != nil {
            return nil, fmt.Errorf("get user %d: context cancelled: %w", id, ctx.Err())
        }
        return nil, fmt.Errorf("get user %d: %w", id, err)
    }
    return &u, nil
}

func (s *UserStore) ListUsers(ctx context.Context, limit, offset int) ([]*User, error) {
    rows, err := s.db.QueryContext(ctx,
        "SELECT id, name, email FROM users ORDER BY id LIMIT $1 OFFSET $2",
        limit, offset,
    )
    if err != nil {
        return nil, fmt.Errorf("list users: %w", err)
    }
    defer rows.Close()

    var users []*User
    for rows.Next() {
        u := &User{}
        if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
            return nil, fmt.Errorf("scan user row: %w", err)
        }
        users = append(users, u)
    }
    return users, rows.Err()
}
```

---

**Scenario 14: Implement structured logging with `log/slog` in a web service.**

```go
package main

import (
    "context"
    "log/slog"
    "net/http"
    "os"
    "time"
)

type contextKey string
const loggerKey contextKey = "logger"

func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func LoggerFromCtx(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return l
    }
    return slog.Default()
}

// RequestLogMiddleware adds a request-scoped logger to context.
func RequestLogMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            reqLogger := logger.With(
                "method", r.Method,
                "path", r.URL.Path,
                "remote_addr", r.RemoteAddr,
            )
            ctx := WithLogger(r.Context(), reqLogger)
            start := time.Now()
            next.ServeHTTP(w, r.WithContext(ctx))
            reqLogger.Info("request completed", "duration_ms", time.Since(start).Milliseconds())
        })
    }
}

func main() {
    logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level:     slog.LevelInfo,
        AddSource: true,
    }))
    slog.SetDefault(logger)

    mux := http.NewServeMux()
    mux.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
        log := LoggerFromCtx(r.Context())
        log.Info("handling users request")
        // handler logic...
        w.WriteHeader(http.StatusOK)
    })

    handler := RequestLogMiddleware(logger)(mux)
    http.ListenAndServe(":8080", handler)
}
```

---

**Scenario 15: Use `io.TeeReader` to simultaneously read and compute a checksum.**

```go
package main

import (
    "crypto/sha256"
    "fmt"
    "io"
    "net/http"
    "os"
)

func downloadAndVerify(url, expectedSHA256, destPath string) error {
    resp, err := http.Get(url)
    if err != nil {
        return fmt.Errorf("download %s: %w", url, err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("download %s: HTTP %d", url, resp.StatusCode)
    }

    f, err := os.Create(destPath)
    if err != nil {
        return fmt.Errorf("create %s: %w", destPath, err)
    }
    defer f.Close()

    // Compute hash while writing to file — single read pass
    h := sha256.New()
    tee := io.TeeReader(resp.Body, h) // everything read from tee also written to h

    if _, err := io.Copy(f, tee); err != nil {
        return fmt.Errorf("write to %s: %w", destPath, err)
    }

    actual := fmt.Sprintf("%x", h.Sum(nil))
    if actual != expectedSHA256 {
        os.Remove(destPath) // clean up corrupt file
        return fmt.Errorf("checksum mismatch: got %s, want %s", actual, expectedSHA256)
    }

    return nil
}
```

---

**Scenario 16: Use `bufio.Scanner` with a custom split function to parse a binary protocol.**

```go
package main

import (
    "bufio"
    "bytes"
    "encoding/binary"
    "fmt"
    "io"
)

// Protocol: each message is prefixed with a 4-byte big-endian length.
func messageScanner(r io.Reader) *bufio.Scanner {
    scanner := bufio.NewScanner(r)
    scanner.Split(func(data []byte, atEOF bool) (advance int, token []byte, err error) {
        if atEOF && len(data) == 0 {
            return 0, nil, nil
        }
        // Need at least 4 bytes for the length prefix
        if len(data) < 4 {
            return 0, nil, nil // request more data
        }

        msgLen := int(binary.BigEndian.Uint32(data[:4]))
        totalLen := 4 + msgLen

        if len(data) < totalLen {
            return 0, nil, nil // request more data
        }

        return totalLen, data[4:totalLen], nil
    })
    scanner.Buffer(make([]byte, 1<<20), 1<<20) // 1MB buffer
    return scanner
}

func processMessages(r io.Reader) error {
    scanner := messageScanner(r)
    for scanner.Scan() {
        msg := scanner.Bytes()
        fmt.Printf("message (%d bytes): %x\n", len(msg), msg)
    }
    return scanner.Err()
}

// Encode a message
func encodeMessage(payload []byte) []byte {
    buf := &bytes.Buffer{}
    binary.Write(buf, binary.BigEndian, uint32(len(payload)))
    buf.Write(payload)
    return buf.Bytes()
}
```

---

**Scenario 17: How do you pipe two commands together using `io.Pipe`?**

```go
package main

import (
    "bufio"
    "fmt"
    "io"
    "strings"
)

// Simulate: grep "error" | wc -l using io.Pipe

func filterLines(r io.Reader, pattern string) io.Reader {
    pr, pw := io.Pipe()

    go func() {
        defer pw.Close()
        scanner := bufio.NewScanner(r)
        for scanner.Scan() {
            line := scanner.Text()
            if strings.Contains(line, pattern) {
                fmt.Fprintln(pw, line)
            }
        }
        if err := scanner.Err(); err != nil {
            pw.CloseWithError(err)
        }
    }()

    return pr
}

func countLines(r io.Reader) (int, error) {
    scanner := bufio.NewScanner(r)
    count := 0
    for scanner.Scan() {
        count++
    }
    return count, scanner.Err()
}

func main() {
    logData := `INFO: server started
ERROR: connection refused
INFO: request received
ERROR: database timeout
WARN: cache miss`

    filtered := filterLines(strings.NewReader(logData), "ERROR")
    count, err := countLines(filtered)
    if err != nil {
        fmt.Println("error:", err)
        return
    }
    fmt.Printf("found %d error lines\n", count) // 2
}
```

---

**Scenario 18: How do you serve static files and embed them in the binary?**

```go
package main

import (
    "embed"
    "io/fs"
    "net/http"
)

//go:embed static/*
var staticFiles embed.FS

func main() {
    // Serve embedded files
    staticFS, err := fs.Sub(staticFiles, "static")
    if err != nil {
        panic(err)
    }

    mux := http.NewServeMux()

    // Serve embedded static files at /static/
    mux.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(staticFS))))

    // API route
    mux.HandleFunc("/api/health", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`{"status":"ok"}`))
    })

    http.ListenAndServe(":8080", mux)
}
```

The `//go:embed` directive embeds the entire `static/` directory into the binary at compile time.
