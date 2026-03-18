# Go Error Handling — Scenario Questions

## File Parsing and Custom Errors

**Scenario 1: You're building a CSV file parser. Design an error type that includes the line number and file name.**

How would you structure the error, and how would callers use it?

**Expected answer:**

```go
package parser

import "fmt"

// ParseError carries the context needed to locate an error in a file.
type ParseError struct {
    File    string
    Line    int
    Column  int
    Message string
    Err     error // underlying cause, if any
}

func (e *ParseError) Error() string {
    if e.Column > 0 {
        return fmt.Sprintf("%s:%d:%d: %s", e.File, e.Line, e.Column, e.Message)
    }
    return fmt.Sprintf("%s:%d: %s", e.File, e.Line, e.Message)
}

func (e *ParseError) Unwrap() error {
    return e.Err
}

// Usage in parser
func ParseCSV(filename string, r io.Reader) ([][]string, error) {
    scanner := bufio.NewScanner(r)
    lineNum := 0
    var records [][]string

    for scanner.Scan() {
        lineNum++
        line := scanner.Text()
        fields, err := parseCSVLine(line)
        if err != nil {
            return nil, &ParseError{
                File:    filename,
                Line:    lineNum,
                Message: "malformed CSV line",
                Err:     err,
            }
        }
        records = append(records, fields)
    }
    if err := scanner.Err(); err != nil {
        return nil, &ParseError{File: filename, Line: lineNum, Message: "scanner error", Err: err}
    }
    return records, nil
}

// Caller can extract structured info
func main() {
    records, err := ParseCSV("data.csv", f)
    if err != nil {
        var pe *ParseError
        if errors.As(err, &pe) {
            fmt.Printf("parse failed at %s line %d\n", pe.File, pe.Line)
        }
        log.Fatal(err)
    }
}
```

Key points: implement `Error() string` and `Unwrap() error`; use `errors.As` to extract the typed error in the caller.

---

**Scenario 2: Your parser encounters an unexpected token. How do you communicate both the expected and actual token?**

```go
type TokenError struct {
    Line     int
    Expected string
    Got      string
}

func (e *TokenError) Error() string {
    return fmt.Sprintf("line %d: expected %s, got %q", e.Line, e.Expected, e.Got)
}

func parseStatement(tokens []Token, line int) error {
    if tokens[0].Type != TokenKeyword {
        return &TokenError{
            Line:     line,
            Expected: "keyword (SELECT, INSERT, UPDATE, DELETE)",
            Got:      tokens[0].Value,
        }
    }
    return nil
}
```

---

## Multi-Service Error Aggregation

**Scenario 3: A function calls three independent services. If any fail, return all errors, not just the first. How do you implement this?**

This is a common pattern when you want maximum information from a batch operation.

```go
import (
    "errors"
    "fmt"
    "strings"
    "sync"
)

type MultiError struct {
    mu   sync.Mutex
    errs []error
}

func (m *MultiError) Append(err error) {
    if err == nil {
        return
    }
    m.mu.Lock()
    m.errs = append(m.errs, err)
    m.mu.Unlock()
}

func (m *MultiError) Error() string {
    m.mu.Lock()
    defer m.mu.Unlock()
    msgs := make([]string, len(m.errs))
    for i, e := range m.errs {
        msgs[i] = fmt.Sprintf("[%d] %s", i+1, e.Error())
    }
    return strings.Join(msgs, "; ")
}

func (m *MultiError) Unwrap() []error {
    return m.errs
}

func (m *MultiError) OrNil() error {
    m.mu.Lock()
    defer m.mu.Unlock()
    if len(m.errs) == 0 {
        return nil
    }
    return m
}

func notifyAllChannels(userID int) error {
    var (
        wg   sync.WaitGroup
        merr MultiError
    )

    wg.Add(3)

    go func() {
        defer wg.Done()
        if err := sendEmail(userID); err != nil {
            merr.Append(fmt.Errorf("email service: %w", err))
        }
    }()

    go func() {
        defer wg.Done()
        if err := sendSMS(userID); err != nil {
            merr.Append(fmt.Errorf("sms service: %w", err))
        }
    }()

    go func() {
        defer wg.Done()
        if err := sendPushNotification(userID); err != nil {
            merr.Append(fmt.Errorf("push service: %w", err))
        }
    }()

    wg.Wait()
    return merr.OrNil()
}
```

---

## Database Error Differentiation

**Scenario 4: You need to distinguish between a "not found" error and a general DB error in your service layer. How do you design this?**

The problem: `database/sql` returns `sql.ErrNoRows` for missing rows, but your service layer should not leak database types to the HTTP handler layer.

```go
// domain/errors.go — domain-layer sentinels
package domain

import "errors"

var (
    ErrNotFound     = errors.New("not found")
    ErrConflict     = errors.New("conflict")
    ErrUnauthorized = errors.New("unauthorized")
)

// store/user_store.go — translate DB errors to domain errors
package store

import (
    "database/sql"
    "errors"
    "fmt"
    "myapp/domain"
)

func (s *UserStore) GetByID(id int64) (*domain.User, error) {
    var u domain.User
    err := s.db.QueryRow("SELECT id, name, email FROM users WHERE id = $1", id).
        Scan(&u.ID, &u.Name, &u.Email)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            // Translate to domain error — callers only need to know domain.ErrNotFound
            return nil, fmt.Errorf("user %d: %w", id, domain.ErrNotFound)
        }
        return nil, fmt.Errorf("get user %d: %w", id, err)
    }
    return &u, nil
}

// handler/user_handler.go — use domain errors
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    id := parseID(r)
    user, err := h.store.GetByID(id)
    if err != nil {
        switch {
        case errors.Is(err, domain.ErrNotFound):
            http.Error(w, "user not found", http.StatusNotFound)
        case errors.Is(err, domain.ErrUnauthorized):
            http.Error(w, "unauthorized", http.StatusUnauthorized)
        default:
            log.Printf("get user: %v", err)
            http.Error(w, "internal error", http.StatusInternalServerError)
        }
        return
    }
    json.NewEncoder(w).Encode(user)
}
```

The key insight: the `%w` wrapping preserves `domain.ErrNotFound` even though the full message has context. `errors.Is` traverses the chain.

---

## Recovering from Third-Party Panics

**Scenario 5: A third-party library panics on invalid input. You cannot modify the library. How do you safely call it?**

```go
package safe

import (
    "fmt"
    "runtime/debug"
)

// SafeCall wraps any function call and converts panics to errors.
func SafeCall(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            switch v := r.(type) {
            case error:
                err = fmt.Errorf("panic recovered: %w", v)
            case string:
                err = fmt.Errorf("panic recovered: %s", v)
            default:
                err = fmt.Errorf("panic recovered: %v", v)
            }
            // Optional: include stack trace
            err = fmt.Errorf("%w\nstack:\n%s", err, debug.Stack())
        }
    }()
    fn()
    return nil
}

// Usage
func parseDocument(data []byte) (doc *Document, err error) {
    var result *thirdparty.Doc
    err = SafeCall(func() {
        result = thirdparty.Parse(data) // may panic on malformed input
    })
    if err != nil {
        return nil, fmt.Errorf("parseDocument: %w", err)
    }
    return convertDoc(result), nil
}
```

---

**Scenario 6: You're writing a concurrent job processor. Each job can fail independently. How do you collect all job errors without stopping other jobs?**

```go
package processor

import (
    "context"
    "errors"
    "fmt"
    "sync"
)

type JobResult struct {
    JobID int
    Err   error
}

func ProcessJobs(ctx context.Context, jobs []Job) error {
    results := make(chan JobResult, len(jobs))
    var wg sync.WaitGroup

    for _, job := range jobs {
        wg.Add(1)
        go func(j Job) {
            defer wg.Done()
            defer func() {
                if r := recover(); r != nil {
                    results <- JobResult{
                        JobID: j.ID,
                        Err:   fmt.Errorf("job %d panicked: %v", j.ID, r),
                    }
                }
            }()
            if err := processJob(ctx, j); err != nil {
                results <- JobResult{JobID: j.ID, Err: fmt.Errorf("job %d: %w", j.ID, err)}
            } else {
                results <- JobResult{JobID: j.ID}
            }
        }(job)
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    var errs []error
    for res := range results {
        if res.Err != nil {
            errs = append(errs, res.Err)
        }
    }

    return errors.Join(errs...)
}
```

---

## Error Wrapping Chain Traversal

**Scenario 7: You have a deeply wrapped error. How do you check if it originated from a specific sentinel at any level of the chain?**

```go
var ErrRateLimit = errors.New("rate limit exceeded")

// Errors get wrapped at multiple layers
func callExternalAPI(endpoint string) error {
    return fmt.Errorf("POST %s: %w", endpoint, ErrRateLimit)
}

func syncUser(userID int64) error {
    if err := callExternalAPI("/users"); err != nil {
        return fmt.Errorf("syncUser %d: %w", userID, err)
    }
    return nil
}

func batchSync(userIDs []int64) error {
    for _, id := range userIDs {
        if err := syncUser(id); err != nil {
            return fmt.Errorf("batchSync: %w", err)
        }
    }
    return nil
}

// Handler
err := batchSync(ids)
// err = "batchSync: syncUser 42: POST /users: rate limit exceeded"

if errors.Is(err, ErrRateLimit) {
    // Rate limit! Implement backoff.
    time.Sleep(retryDelay)
} else if err != nil {
    log.Printf("unexpected error: %v", err)
}
```

`errors.Is` walks the entire chain: `batchSync error → syncUser error → callExternalAPI error → ErrRateLimit`. It finds the sentinel at any depth.

---

**Scenario 8: How do you extract a specific error type from a deeply wrapped chain?**

```go
type RateLimitError struct {
    RetryAfter time.Duration
}

func (e *RateLimitError) Error() string {
    return fmt.Sprintf("rate limited, retry after %s", e.RetryAfter)
}

func callAPI() error {
    return fmt.Errorf("api call: %w", &RateLimitError{RetryAfter: 30 * time.Second})
}

func processRequest() error {
    return fmt.Errorf("processRequest: %w", callAPI())
}

func handler() {
    err := processRequest()

    var rle *RateLimitError
    if errors.As(err, &rle) {
        // Got the typed error even through multiple wrapping layers
        fmt.Printf("retry after: %s\n", rle.RetryAfter)
        return
    }
    log.Printf("unexpected error: %v", err)
}
```

---

## Defer and Recover Patterns

**Scenario 9: You are writing a web framework. How do you prevent a panicking handler from crashing the entire server?**

```go
package middleware

import (
    "fmt"
    "log"
    "net/http"
    "runtime/debug"
)

// RecoveryMiddleware catches panics in HTTP handlers and returns 500.
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                // Log the panic with stack trace for debugging
                log.Printf("PANIC recovered: %v\n%s", rec, debug.Stack())

                // Respond with 500 if headers haven't been sent yet
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// Usage
mux := http.NewServeMux()
mux.HandleFunc("/api/data", dataHandler)
http.ListenAndServe(":8080", RecoveryMiddleware(mux))
```

---

**Scenario 10: In a defer, how do you modify the named return value to return an error from a recovery?**

```go
// Named return values allow defer to modify what gets returned
func riskyOperation() (result string, err error) {
    defer func() {
        if r := recover(); r != nil {
            // Assign to the named return 'err'
            err = fmt.Errorf("riskyOperation panicked: %v", r)
            result = "" // clear any partial result
        }
    }()

    // This panics if input is invalid
    result = unsafeStringOperation()
    return result, nil
}

func main() {
    res, err := riskyOperation()
    if err != nil {
        fmt.Println("error:", err) // handled gracefully
    } else {
        fmt.Println("result:", res)
    }
}
```

---

## Validation Error Scenarios

**Scenario 11: You're building a REST API. How do you return all validation errors for a request body at once, not just the first?**

```go
type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

type ValidationErrors struct {
    Errors []FieldError `json:"errors"`
}

func (v *ValidationErrors) Add(field, message string) {
    v.Errors = append(v.Errors, FieldError{Field: field, Message: message})
}

func (v *ValidationErrors) Error() string {
    return fmt.Sprintf("%d validation errors", len(v.Errors))
}

func (v *ValidationErrors) HasErrors() bool {
    return len(v.Errors) > 0
}

func validateCreateUserRequest(req CreateUserRequest) error {
    var ve ValidationErrors

    if req.Name == "" {
        ve.Add("name", "name is required")
    } else if len(req.Name) > 100 {
        ve.Add("name", "name must be 100 characters or fewer")
    }

    if req.Email == "" {
        ve.Add("email", "email is required")
    } else if !isValidEmail(req.Email) {
        ve.Add("email", "email format is invalid")
    }

    if req.Password == "" {
        ve.Add("password", "password is required")
    } else if len(req.Password) < 8 {
        ve.Add("password", "password must be at least 8 characters")
    }

    if ve.HasErrors() {
        return &ve
    }
    return nil
}

// Handler returns all validation errors as JSON
func createUserHandler(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    json.NewDecoder(r.Body).Decode(&req)

    if err := validateCreateUserRequest(req); err != nil {
        var ve *ValidationErrors
        if errors.As(err, &ve) {
            w.Header().Set("Content-Type", "application/json")
            w.WriteHeader(http.StatusUnprocessableEntity)
            json.NewEncoder(w).Encode(ve)
            return
        }
        http.Error(w, "bad request", http.StatusBadRequest)
        return
    }
    // proceed...
}
```

---

**Scenario 12: A database transaction wraps multiple operations. How do you ensure the transaction is rolled back on any error?**

```go
func (s *Store) TransferFunds(ctx context.Context, fromID, toID int64, amount decimal.Decimal) error {
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }

    // Always attempt rollback in defer.
    // If Commit succeeds, Rollback is a no-op on a committed transaction.
    defer func() {
        if rbErr := tx.Rollback(); rbErr != nil && !errors.Is(rbErr, sql.ErrTxDone) {
            log.Printf("rollback error: %v", rbErr)
        }
    }()

    if err := debit(ctx, tx, fromID, amount); err != nil {
        return fmt.Errorf("debit account %d: %w", fromID, err)
    }

    if err := credit(ctx, tx, toID, amount); err != nil {
        return fmt.Errorf("credit account %d: %w", toID, err)
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }

    return nil
}
```

---

**Scenario 13: How do you handle errors in a `goroutine` and propagate them back to the main goroutine?**

```go
func fetchURLs(urls []string) ([]string, error) {
    type result struct {
        url  string
        body string
        err  error
    }

    ch := make(chan result, len(urls))

    for _, url := range urls {
        go func(u string) {
            resp, err := http.Get(u)
            if err != nil {
                ch <- result{url: u, err: fmt.Errorf("GET %s: %w", u, err)}
                return
            }
            defer resp.Body.Close()
            body, err := io.ReadAll(resp.Body)
            if err != nil {
                ch <- result{url: u, err: fmt.Errorf("read body %s: %w", u, err)}
                return
            }
            ch <- result{url: u, body: string(body)}
        }(url)
    }

    var (
        bodies []string
        errs   []error
    )
    for range urls {
        res := <-ch
        if res.err != nil {
            errs = append(errs, res.err)
        } else {
            bodies = append(bodies, res.body)
        }
    }

    return bodies, errors.Join(errs...)
}
```

---

**Scenario 14: How do you safely close resources and still surface the close error without masking an existing error?**

```go
// closeErr pattern: return the close error only if no prior error exists
func writeFile(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return fmt.Errorf("create %s: %w", path, err)
    }

    defer func() {
        closeErr := f.Close()
        if err == nil && closeErr != nil {
            // No prior error — surface the close error
            err = fmt.Errorf("close %s: %w", path, closeErr)
        } else if closeErr != nil {
            // Prior error exists — log the close error, keep the original
            log.Printf("close %s after error: %v", path, closeErr)
        }
    }()

    if _, err = f.Write(data); err != nil {
        return fmt.Errorf("write %s: %w", path, err)
    }
    return nil
}
```

---

**Scenario 15: How do you implement a retry mechanism that distinguishes between retryable and non-retryable errors?**

```go
type RetryableError struct {
    Err error
}

func (e *RetryableError) Error() string { return e.Err.Error() }
func (e *RetryableError) Unwrap() error { return e.Err }

func isRetryable(err error) bool {
    var re *RetryableError
    return errors.As(err, &re)
}

func callWithRetry(ctx context.Context, fn func(context.Context) error, maxAttempts int) error {
    var lastErr error
    for attempt := 0; attempt < maxAttempts; attempt++ {
        if attempt > 0 {
            backoff := time.Duration(attempt) * 500 * time.Millisecond
            select {
            case <-time.After(backoff):
            case <-ctx.Done():
                return fmt.Errorf("retry cancelled: %w", ctx.Err())
            }
        }

        if err := fn(ctx); err != nil {
            lastErr = err
            if !isRetryable(err) {
                return err // Non-retryable: fail immediately
            }
            continue // Retryable: try again
        }
        return nil // Success
    }
    return fmt.Errorf("all %d attempts failed, last error: %w", maxAttempts, lastErr)
}

// Service wraps transient errors as retryable
func fetchData(ctx context.Context) error {
    resp, err := http.Get("https://api.example.com/data")
    if err != nil {
        return &RetryableError{Err: fmt.Errorf("http get: %w", err)}
    }
    if resp.StatusCode == http.StatusServiceUnavailable {
        return &RetryableError{Err: fmt.Errorf("service unavailable (503)")}
    }
    // ...
    return nil
}
```

---

**Scenario 16: You're implementing a CLI tool. How do you provide user-friendly error messages while preserving technical details for logs?**

```go
type UserError struct {
    UserMessage string // shown to the user
    Detail      error  // logged internally
}

func (e *UserError) Error() string { return e.UserMessage }
func (e *UserError) Unwrap() error  { return e.Detail }

func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        if os.IsNotExist(err) {
            return nil, &UserError{
                UserMessage: fmt.Sprintf("Configuration file not found at %q. Run 'myapp init' to create it.", path),
                Detail:      fmt.Errorf("readFile %s: %w", path, err),
            }
        }
        return nil, &UserError{
            UserMessage: "Cannot read configuration file. Check file permissions.",
            Detail:      fmt.Errorf("readFile %s: %w", path, err),
        }
    }
    // ...
}

func main() {
    cfg, err := loadConfig(configPath)
    if err != nil {
        var ue *UserError
        if errors.As(err, &ue) {
            fmt.Fprintln(os.Stderr, "Error:", ue.UserMessage)
        } else {
            fmt.Fprintln(os.Stderr, "Error:", err)
        }
        // Log technical details
        log.Printf("fatal: %+v", err)
        os.Exit(1)
    }
    _ = cfg
}
```

---

**Scenario 17: How do you test that your function returns the correct error type?**

```go
func TestGetUser_NotFound(t *testing.T) {
    store := NewInMemoryStore()
    _, err := store.GetUser(context.Background(), 999)

    // Test 1: error is not nil
    if err == nil {
        t.Fatal("expected error, got nil")
    }

    // Test 2: it wraps the domain sentinel
    if !errors.Is(err, domain.ErrNotFound) {
        t.Errorf("expected domain.ErrNotFound, got: %v", err)
    }

    // Test 3: the full chain contains expected context
    if !strings.Contains(err.Error(), "999") {
        t.Errorf("error message should contain user ID, got: %v", err)
    }
}

func TestGetUser_DatabaseError(t *testing.T) {
    store := NewStoreWithBrokenDB()
    _, err := store.GetUser(context.Background(), 1)

    if errors.Is(err, domain.ErrNotFound) {
        t.Error("expected a db error, not ErrNotFound")
    }
    if err == nil {
        t.Fatal("expected error, got nil")
    }
}
```

---

**Scenario 18: How do you handle errors when using `errors.Join` with Go 1.20+ and iterate over them?**

```go
import (
    "errors"
    "fmt"
)

func validateConfig(cfg Config) error {
    var errs []error

    if cfg.Host == "" {
        errs = append(errs, errors.New("host is required"))
    }
    if cfg.Port == 0 {
        errs = append(errs, errors.New("port is required"))
    }
    if cfg.Timeout <= 0 {
        errs = append(errs, errors.New("timeout must be positive"))
    }

    return errors.Join(errs...)
}

// Unwrap the joined errors
func printAllErrors(err error) {
    type unwrapMulti interface {
        Unwrap() []error
    }

    if mErr, ok := err.(unwrapMulti); ok {
        for i, e := range mErr.Unwrap() {
            fmt.Printf("  error %d: %v\n", i+1, e)
        }
    } else {
        fmt.Printf("  error: %v\n", err)
    }
}

func main() {
    cfg := Config{} // all zero values
    err := validateConfig(cfg)
    if err != nil {
        fmt.Println("Validation failed:")
        printAllErrors(err)
    }
}
// Output:
// Validation failed:
//   error 1: host is required
//   error 2: port is required
//   error 3: timeout must be positive
```
