# Go Error Handling — Structured Answers

## Answer 1: What is the `error` interface and how do you implement a custom error type?

**The `error` interface:**

```go
type error interface {
    Error() string
}
```

It is the simplest possible interface — just one method that returns a string. Any type with this method is an `error`. The standard library provides two ways to create simple errors:

```go
// errors.New — creates a new error with a static message
var ErrNotFound = errors.New("not found")

// fmt.Errorf — creates a formatted error; %w also wraps for chain traversal
return fmt.Errorf("query user %d: %w", id, err)
```

**Implementing a custom error type:**

```go
// Step 1: define a struct with the fields you need
type DBError struct {
    Op      string // operation that failed (e.g., "QueryUser")
    Table   string // table involved
    Code    int    // database error code
    Err     error  // underlying error
}

// Step 2: implement Error() string
func (e *DBError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("db %s on %s (code %d): %v", e.Op, e.Table, e.Code, e.Err)
    }
    return fmt.Sprintf("db %s on %s (code %d)", e.Op, e.Table, e.Code)
}

// Step 3: implement Unwrap() error if you hold a cause
func (e *DBError) Unwrap() error {
    return e.Err
}

// Step 4: use errors.As to extract from callers
func getUserByEmail(db *sql.DB, email string) (*User, error) {
    var u User
    err := db.QueryRow("SELECT id, name FROM users WHERE email = $1", email).Scan(&u.ID, &u.Name)
    if err != nil {
        return nil, &DBError{Op: "QueryRow", Table: "users", Code: pgErrCode(err), Err: err}
    }
    return &u, nil
}

// Caller
user, err := getUserByEmail(db, "alice@example.com")
if err != nil {
    var dbe *DBError
    if errors.As(err, &dbe) {
        fmt.Printf("failed operation: %s, code: %d\n", dbe.Op, dbe.Code)
    }
    return err
}
```

**Key rules:**
- Use a pointer receiver (`*DBError`) so the type satisfies `error` consistently
- Implement `Unwrap()` whenever you hold an underlying cause
- Export the type if callers need to use `errors.As`

---

## Answer 2: What is the difference between `errors.Is()` and `errors.As()`?

Both traverse the error chain by repeatedly calling `Unwrap()`, but they check for different things:

| Function | Checks | Use when |
|---|---|---|
| `errors.Is(err, target)` | Identity equality | You want to know if a **specific sentinel** is in the chain |
| `errors.As(err, &target)` | Type compatibility | You want to **extract a typed error** and read its fields |

```go
var ErrTimeout = errors.New("timeout")

type TimeoutError struct {
    Duration time.Duration
    Op       string
}
func (e *TimeoutError) Error() string {
    return fmt.Sprintf("%s timed out after %s", e.Op, e.Duration)
}

// --- errors.Is ---
// Checks if any error in the chain IS (equals) ErrTimeout
err1 := fmt.Errorf("connect: %w", ErrTimeout)
fmt.Println(errors.Is(err1, ErrTimeout))   // true
fmt.Println(errors.Is(err1, io.EOF))        // false

// --- errors.As ---
// Checks if any error in the chain can be assigned to *TimeoutError
err2 := fmt.Errorf("connect: %w", &TimeoutError{Duration: 5 * time.Second, Op: "dial"})
var te *TimeoutError
if errors.As(err2, &te) {
    fmt.Printf("operation %q timed out after %s\n", te.Op, te.Duration)
}
```

**errors.Is custom matching:**
A type can override the default `==` check by implementing `Is(target error) bool`:

```go
type StatusError struct{ Code int }
func (e *StatusError) Error() string { return fmt.Sprintf("HTTP %d", e.Code) }
func (e *StatusError) Is(target error) bool {
    t, ok := target.(*StatusError)
    return ok && t.Code == e.Code
}

var Err404 = &StatusError{Code: 404}
wrapped := fmt.Errorf("handler: %w", &StatusError{Code: 404})
fmt.Println(errors.Is(wrapped, Err404)) // true — uses custom Is()
```

---

## Answer 3: How does error wrapping work with `%w` and `errors.Unwrap()`?

`fmt.Errorf` with `%w` creates a new error that:
1. Has a combined message (the format string + wrapped error's message)
2. Implements `Unwrap() error` returning the wrapped error
3. Is traversable by `errors.Is` and `errors.As`

```go
// Building a chain
base := errors.New("connection refused")
layer1 := fmt.Errorf("dial tcp 127.0.0.1:5432: %w", base)
layer2 := fmt.Errorf("open database: %w", layer1)
layer3 := fmt.Errorf("initialize app: %w", layer2)

// Messages compose naturally
fmt.Println(layer3)
// initialize app: open database: dial tcp 127.0.0.1:5432: connection refused

// Unwrap manually
fmt.Println(errors.Unwrap(layer3)) // open database: dial tcp...
fmt.Println(errors.Unwrap(errors.Unwrap(layer3))) // dial tcp...
fmt.Println(errors.Unwrap(errors.Unwrap(errors.Unwrap(layer3)))) // connection refused
fmt.Println(errors.Unwrap(base)) // <nil>

// errors.Is traverses the entire chain
fmt.Println(errors.Is(layer3, base)) // true
```

**Implementing Unwrap manually on a custom type:**

```go
type OperationError struct {
    Op  string
    Err error
}

func (e *OperationError) Error() string {
    return e.Op + ": " + e.Err.Error()
}

func (e *OperationError) Unwrap() error {
    return e.Err
}
```

**Multiple wrapping (Go 1.20+):**

```go
err1 := errors.New("err1")
err2 := errors.New("err2")
joined := errors.Join(err1, err2) // implements Unwrap() []error

fmt.Println(errors.Is(joined, err1)) // true
fmt.Println(errors.Is(joined, err2)) // true
```

---

## Answer 4: When should you use `panic` vs returning an error?

**Return an error when:**
- The condition is expected and the caller should handle it (file not found, network timeout, invalid input, DB constraint violation)
- You are writing a library — libraries should never panic on expected conditions
- Any production code that handles normal operating conditions

**Use `panic` when:**
- A programming invariant has been violated and continuing would be unsafe (nil pointer on a required dependency, impossible state, index out of bounds on internal data)
- A `Must*` wrapper function is appropriate (e.g., `regexp.MustCompile`, `template.Must`)
- Initialization code that cannot sensibly recover (startup configuration is completely missing)

```go
// Appropriate: Must* functions panic on programmer error
var validEmail = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

// Appropriate: init-time invariant
func New(cfg Config) *Service {
    if cfg.DB == nil {
        panic("service: DB is required") // programming error — caller must provide DB
    }
    return &Service{db: cfg.DB}
}

// NOT appropriate: runtime condition — return error
func fetchUser(id int) (*User, error) {
    if id <= 0 {
        return nil, fmt.Errorf("fetchUser: id must be positive, got %d", id)
    }
    // ...
}
```

**Go philosophy:** Panic is for programmers. Errors are for callers.

---

## Answer 5: How do you recover from a panic and convert it to an error?

The canonical pattern uses a deferred function with named return values:

```go
// Pattern 1: named returns — defer can modify what gets returned
func safeExecute(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            switch v := r.(type) {
            case error:
                err = fmt.Errorf("panic: %w", v)
            default:
                err = fmt.Errorf("panic: %v", v)
            }
        }
    }()
    fn()
    return nil
}

// Pattern 2: wrapping a specific third-party call
func safeThirdPartyParse(data []byte) (doc *Document, err error) {
    defer func() {
        if r := recover(); r != nil {
            doc = nil
            err = fmt.Errorf("thirdparty.Parse panicked on input (len=%d): %v", len(data), r)
        }
    }()
    doc = thirdparty.Parse(data) // may panic
    return doc, nil
}

// Pattern 3: generic middleware — convert all panics in an HTTP handler
func withRecovery(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                log.Printf("recovered panic: %v\n%s", rec, debug.Stack())
                http.Error(w, "internal server error", http.StatusInternalServerError)
            }
        }()
        h.ServeHTTP(w, r)
    })
}
```

**Critical rules:**
1. `recover()` must be called **directly** inside a `defer` function
2. `recover()` only works within the **same goroutine** as the panic
3. After recovery, the panicking goroutine's stack is gone — only deferred functions ran

---

## Answer 6: What are sentinel errors and what are their downsides?

**Sentinel errors** are package-level variables that represent specific error conditions. They are the Go equivalent of well-known error codes:

```go
package store

import "errors"

// Sentinels — exported so callers can compare
var (
    ErrNotFound    = errors.New("not found")
    ErrConflict    = errors.New("conflict")
    ErrInvalidInput = errors.New("invalid input")
)

// Usage
func (s *Store) GetUser(id int64) (*User, error) {
    // ...
    if notFound {
        return nil, fmt.Errorf("user %d: %w", id, ErrNotFound)
    }
}

// Caller
if errors.Is(err, store.ErrNotFound) {
    return http.StatusNotFound
}
```

**Pros:**
- Simple, explicit API contract
- Stable identities that `errors.Is` can find through wrapping
- Self-documenting code

**Downsides:**
- They become part of the public API — removing them is a breaking change
- They carry no contextual data (just identity, no "which user was not found")
- Importing them creates a package dependency
- They encourage callers to write `switch` statements on sentinel values, creating tight coupling

**Alternative for richer data:** use custom error types with `errors.As` instead of sentinels with `errors.Is`:

```go
// More expressive than a sentinel
type NotFoundError struct {
    Resource string
    ID       interface{}
}
func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s with id %v not found", e.Resource, e.ID)
}

// Caller gets the resource type and ID
var nfe *NotFoundError
if errors.As(err, &nfe) {
    log.Printf("%s %v was not found", nfe.Resource, nfe.ID)
}
```

---

## Answer 7: How do you accumulate multiple errors (multierror pattern)?

When operations are independent and you want to collect all failures before returning:

```go
package errs

import (
    "errors"
    "strings"
)

// MultiError collects multiple errors.
type MultiError struct {
    errs []error
}

func (m *MultiError) Add(err error) {
    if err != nil {
        m.errs = append(m.errs, err)
    }
}

func (m *MultiError) Error() string {
    msgs := make([]string, len(m.errs))
    for i, e := range m.errs {
        msgs[i] = e.Error()
    }
    return strings.Join(msgs, "; ")
}

// Unwrap returns all errors so errors.Is/As can traverse them (Go 1.20+)
func (m *MultiError) Unwrap() []error {
    return m.errs
}

// OrNil returns nil if there are no errors, otherwise returns the MultiError.
func (m *MultiError) OrNil() error {
    if len(m.errs) == 0 {
        return nil
    }
    return m
}

// Usage: validate all fields before returning
func validateOrder(order Order) error {
    var me MultiError

    if order.CustomerID == 0 {
        me.Add(errors.New("customer_id is required"))
    }
    if len(order.Items) == 0 {
        me.Add(errors.New("order must have at least one item"))
    }
    for i, item := range order.Items {
        if item.Quantity <= 0 {
            me.Add(fmt.Errorf("item[%d]: quantity must be positive", i))
        }
        if item.Price < 0 {
            me.Add(fmt.Errorf("item[%d]: price cannot be negative", i))
        }
    }

    return me.OrNil()
}

// Go 1.20+ shorthand with errors.Join
func validateOrderSimple(order Order) error {
    var errs []error
    if order.CustomerID == 0 {
        errs = append(errs, errors.New("customer_id required"))
    }
    // ...
    return errors.Join(errs...)
}
```

---

## Answer 8: What is the non-nil nil interface error trap?

This is one of Go's most surprising gotchas. An interface value is nil only when **both** its type and value are nil. If you store a typed nil pointer in an interface, the interface is not nil.

```go
type AppError struct{ Code int }
func (e *AppError) Error() string { return fmt.Sprintf("code %d", e.Code) }

func getError(fail bool) error {
    var err *AppError // (*AppError)(nil) — typed nil
    if fail {
        err = &AppError{Code: 500}
    }
    return err // BUG: interface{type: *AppError, value: nil} — non-nil!
}

func main() {
    err := getError(false)
    if err != nil {
        // This branch IS taken — err is non-nil interface
        fmt.Println("got error:", err) // prints "got error: code 0"
        // Calling err.Error() on a nil pointer method receiver also panics
    }
}
```

**How the interface looks in memory:**

```
getError(false) returns interface{
    type:  *AppError   ← non-nil (has type information)
    value: nil         ← the pointer itself is nil
}

nil error looks like:
interface{
    type:  nil         ← no type
    value: nil         ← no value
}
```

**Fix:** Never return a typed nil as an error:

```go
func getError(fail bool) error {
    if fail {
        return &AppError{Code: 500}
    }
    return nil // untyped nil — safe
}

// Also catches this anti-pattern with linters like errcheck
// Use -S with staticcheck to find these
```

---

## Answer 9: How do you add stack traces to errors in Go?

Go's standard library does **not** capture stack traces in errors. You have three options:

**Option 1: `github.com/pkg/errors` (captures trace at wrap site)**

```go
import "github.com/pkg/errors"

func readConfig(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return errors.Wrap(err, "readConfig") // captures stack trace HERE
    }
    defer f.Close()
    return nil
}

func main() {
    err := readConfig("/missing/config.json")
    fmt.Printf("%+v\n", err)
    // readConfig
    // main.readConfig
    //         /path/to/main.go:10
    // main.main
    //         /path/to/main.go:16
    // open /missing/config.json: no such file or directory
}
```

**Option 2: Custom error with `runtime.Callers`**

```go
import "runtime"

type ErrorWithStack struct {
    err   error
    stack []uintptr
}

func newErrorWithStack(err error) *ErrorWithStack {
    var pcs [32]uintptr
    n := runtime.Callers(2, pcs[:])
    return &ErrorWithStack{err: err, stack: pcs[:n]}
}

func (e *ErrorWithStack) Error() string { return e.err.Error() }
func (e *ErrorWithStack) Unwrap() error { return e.err }

func (e *ErrorWithStack) StackTrace() string {
    frames := runtime.CallersFrames(e.stack)
    var sb strings.Builder
    for {
        frame, more := frames.Next()
        fmt.Fprintf(&sb, "%s\n\t%s:%d\n", frame.Function, frame.File, frame.Line)
        if !more {
            break
        }
    }
    return sb.String()
}
```

**Option 3: Structured logging approach (most common in modern Go)**

Rather than storing stack traces in errors, log the error with a trace at the boundary:

```go
func handler(w http.ResponseWriter, r *http.Request) {
    if err := processRequest(r); err != nil {
        // Log with stack trace at the boundary
        log.Printf("processRequest failed: %v\n%s", err, debug.Stack())
        http.Error(w, "internal error", 500)
    }
}
```

---

## Answer 10: What are Go's best practices for error messages?

**1. Lowercase, no trailing punctuation:**

```go
// Bad
errors.New("File not found.")
fmt.Errorf("Failed to connect to server: %w", err)

// Good
errors.New("file not found")
fmt.Errorf("connect to server: %w", err)
```

**2. Include the operation as a prefix — builds a breadcrumb trail:**

```go
// Each layer adds its own name
func (s *Store) GetUser(id int64) (*User, error) {
    // ...
    return nil, fmt.Errorf("store.GetUser %d: %w", id, err)
}

func (svc *UserService) Profile(id int64) (*Profile, error) {
    u, err := svc.store.GetUser(id)
    if err != nil {
        return nil, fmt.Errorf("UserService.Profile: %w", err)
    }
    // ...
}
// Result: "UserService.Profile: store.GetUser 42: sql: no rows in result set"
```

**3. Include the relevant values (IDs, paths, addresses) in the message:**

```go
// Not enough context
return fmt.Errorf("user not found: %w", err)

// Better — includes the ID
return fmt.Errorf("user %d not found: %w", id, err)

// Best — includes the operation context too
return fmt.Errorf("getUser id=%d: %w", id, err)
```

**4. Don't log AND return — pick one:**

```go
// Anti-pattern: double logging
func doSomething() error {
    if err := step(); err != nil {
        log.Printf("step failed: %v", err) // logs here
        return err                          // caller also logs — duplicate entries
    }
    return nil
}

// Better: only log at the top-level boundary (HTTP handler, main, etc.)
func doSomething() error {
    if err := step(); err != nil {
        return fmt.Errorf("doSomething: %w", err) // wrap and return
    }
    return nil
}
```

**5. Do not expose implementation details in error messages returned to users:**

```go
// Dangerous — leaks DB internals to API response
return fmt.Errorf("pq: duplicate key value violates unique constraint users_email_key")

// Safe — user-facing message; log the technical detail separately
log.Printf("create user: %v", err)
return ErrEmailAlreadyExists
```

**6. Use `%w` not `%v` when wrapping for chain traversal:**

```go
// Breaks errors.Is/errors.As
return fmt.Errorf("context: %v", originalErr)

// Preserves chain for errors.Is/errors.As
return fmt.Errorf("context: %w", originalErr)
```
