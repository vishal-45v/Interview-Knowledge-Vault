# Go Error Handling — Theory Questions

## The `error` Interface

**Q1. What is the `error` interface in Go? Show its definition.**

The `error` interface is a built-in interface with a single method:

```go
type error interface {
    Error() string
}
```

Any type that implements the `Error() string` method satisfies this interface. Because Go uses structural (implicit) typing, there is no `implements` keyword — if your type has that method, it is an `error`.

---

**Q2. Why does Go treat errors as values rather than exceptions?**

Go's design philosophy is that exceptional conditions (network failures, missing files, invalid input) are expected parts of a program's operation, not truly exceptional events. Treating errors as values:

- Makes control flow explicit — you can see every place an error is handled
- Prevents the "invisible goto" problem of exceptions, where control jumps unpredictably
- Forces the programmer to make a deliberate decision at each error site
- Allows errors to be stored, passed around, compared, and manipulated like any other value

Rob Pike's maxim: "Errors are values."

---

**Q3. How do you create a simple error using `errors.New()`?**

```go
import "errors"

var ErrNotFound = errors.New("resource not found")

func findUser(id int) (*User, error) {
    if id <= 0 {
        return nil, errors.New("invalid user ID: must be positive")
    }
    // ...
    return nil, ErrNotFound
}
```

`errors.New` returns a pointer to an unexported `errorString` struct. Each call to `errors.New` creates a distinct error value, so two calls with the same message are not equal via `==`.

---

**Q4. How do you create formatted errors using `fmt.Errorf()`?**

```go
import "fmt"

func openConfig(path string) error {
    return fmt.Errorf("openConfig: cannot open %q: %w", path, err)
}
```

`fmt.Errorf` supports the full `fmt` verb set. The `%w` verb (introduced in Go 1.13) wraps an existing error so it can be retrieved later via `errors.Unwrap()`, `errors.Is()`, and `errors.As()`.

---

**Q5. What is a custom error type and how do you implement one?**

Any struct that implements `Error() string` is a custom error type:

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on field %q: %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 || age > 150 {
        return &ValidationError{Field: "age", Message: "must be between 0 and 150"}
    }
    return nil
}
```

Custom types let callers extract structured information beyond just a string message.

---

## Error Wrapping (Go 1.13+)

**Q6. What is error wrapping and how do you wrap an error with `fmt.Errorf`?**

Error wrapping embeds one error inside another, forming a chain. This preserves the original error while adding context about where in the call stack things went wrong.

```go
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("readConfig %s: %w", path, err)
    }
    // ...
}

func initApp() error {
    cfg, err := readConfig("/etc/app/config.json")
    if err != nil {
        return fmt.Errorf("initApp: %w", err)
    }
    _ = cfg
    return nil
}
```

The resulting error message reads like a call chain: `initApp: readConfig /etc/app/config.json: open /etc/app/config.json: no such file or directory`.

---

**Q7. What does `errors.Unwrap()` do?**

`errors.Unwrap` returns the wrapped error, or `nil` if the error does not implement the `Unwrap() error` method:

```go
err1 := errors.New("original")
err2 := fmt.Errorf("layer 2: %w", err1)
err3 := fmt.Errorf("layer 3: %w", err2)

fmt.Println(errors.Unwrap(err3)) // "layer 2: original"
fmt.Println(errors.Unwrap(err2)) // "original"
fmt.Println(errors.Unwrap(err1)) // <nil>
```

You can implement `Unwrap` on your own types:

```go
type DBError struct {
    Op  string
    Err error
}

func (e *DBError) Error() string { return e.Op + ": " + e.Err.Error() }
func (e *DBError) Unwrap() error  { return e.Err }
```

---

**Q8. What is the difference between `errors.Is()` and `errors.As()`?**

- `errors.Is(err, target)` — checks if any error in the chain **equals** `target`. It uses `==` by default, but a type can override this by implementing `Is(target error) bool`. Used for sentinel errors.
- `errors.As(err, target)` — checks if any error in the chain can be **assigned** to the type pointed to by `target`. Used to extract structured data from a custom error type.

```go
var ErrNotFound = errors.New("not found")

// errors.Is — identity check
wrapped := fmt.Errorf("db query: %w", ErrNotFound)
fmt.Println(errors.Is(wrapped, ErrNotFound)) // true

// errors.As — type extraction
type QueryError struct{ Query string; Err error }
func (e *QueryError) Error() string { return e.Query + ": " + e.Err.Error() }
func (e *QueryError) Unwrap() error { return e.Err }

qErr := &QueryError{Query: "SELECT *", Err: ErrNotFound}
wrapped2 := fmt.Errorf("handler: %w", qErr)

var qe *QueryError
if errors.As(wrapped2, &qe) {
    fmt.Println("failed query:", qe.Query) // "SELECT *"
}
```

---

**Q9. When should you use `errors.Is()` vs direct `==` comparison?**

Always prefer `errors.Is()` when the error may have been wrapped. Direct `==` only matches the exact top-level error; `errors.Is()` traverses the entire chain.

```go
err := fmt.Errorf("context: %w", io.EOF)

// Wrong — misses wrapped errors
if err == io.EOF { /* never true */ }

// Correct — traverses the chain
if errors.Is(err, io.EOF) { /* true */ }
```

---

## Sentinel Errors

**Q10. What are sentinel errors? Give examples from the standard library.**

Sentinel errors are package-level error variables that represent specific, well-known error conditions. Callers compare against them to branch their logic.

```go
// Standard library sentinels
import (
    "io"
    "database/sql"
)

// io.EOF signals end of stream (not truly an error — expected condition)
n, err := reader.Read(buf)
if err == io.EOF { /* done reading */ }

// sql.ErrNoRows signals a query returned nothing
row := db.QueryRow("SELECT id FROM users WHERE email = ?", email)
var id int
if err := row.Scan(&id); errors.Is(err, sql.ErrNoRows) {
    // user doesn't exist
}
```

---

**Q11. What are the pros and cons of sentinel errors?**

**Pros:**
- Explicit, named conditions that callers can document against
- Simple to check with `errors.Is()`
- Create a stable API contract

**Cons:**
- They become part of the package's public API — hard to remove
- They carry no contextual data (just identity)
- They create package-level coupling — importing `database/sql` just to check `sql.ErrNoRows`
- If you wrap them with additional context, callers must use `errors.Is()` not `==`

---

**Q12. How do you define your own sentinel errors?**

```go
package store

import "errors"

// ErrNotFound is returned when a requested resource does not exist.
var ErrNotFound = errors.New("not found")

// ErrConflict is returned when an insert would violate a uniqueness constraint.
var ErrConflict = errors.New("conflict")

// ErrInvalidInput is returned when caller-provided data fails validation.
var ErrInvalidInput = errors.New("invalid input")
```

Convention: name them `ErrXxx`, document them, and export them if callers need to distinguish them.

---

## panic and recover

**Q13. What is `panic()` in Go and when should you use it?**

`panic` terminates the current goroutine's normal execution, runs any deferred functions, and then propagates up the call stack. If it reaches the top of a goroutine without being recovered, it crashes the program with a stack trace.

Use `panic` only for:
1. **Programming errors** that should never occur in correct code (nil pointer dereference on a required dependency, index out of bounds on internal data structures)
2. **Unrecoverable initialization failures** (e.g., a template fails to parse at startup)

Do **not** use `panic` for expected error conditions like file not found, network timeout, or invalid user input. Return an `error` instead.

```go
// Appropriate panic — programmer error
func MustCompile(pattern string) *regexp.Regexp {
    re, err := regexp.Compile(pattern)
    if err != nil {
        panic(fmt.Sprintf("MustCompile: invalid pattern %q: %v", pattern, err))
    }
    return re
}

// Never do this — this is not a panic situation
func readFile(path string) {
    data, err := os.ReadFile(path)
    if err != nil {
        panic(err) // Wrong! Return the error instead.
    }
    _ = data
}
```

---

**Q14. What is `recover()` and how does it work?**

`recover()` stops a panicking goroutine and returns the panic value. It only works when called directly inside a deferred function. Outside a deferred function, `recover()` always returns `nil`.

```go
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered panic: %v", r)
        }
    }()
    result = a / b
    return
}

func main() {
    result, err := safeDiv(10, 0)
    fmt.Println(result, err) // 0 recovered panic: runtime error: integer divide by zero
}
```

---

**Q15. How do you convert a panic from a third-party library into a Go error?**

Wrap the call in a function that defers a recovery:

```go
func safeParse(data []byte) (result *ParsedDoc, err error) {
    defer func() {
        if r := recover(); r != nil {
            // Convert whatever was panicked (string, error, or anything) to error
            switch v := r.(type) {
            case error:
                err = fmt.Errorf("thirdparty.Parse panicked: %w", v)
            default:
                err = fmt.Errorf("thirdparty.Parse panicked: %v", v)
            }
        }
    }()
    result = thirdparty.Parse(data) // may panic
    return
}
```

---

**Q16. Why does `recover()` have to be in a deferred function?**

By the time `panic` is executing, the normal call stack is unwinding — ordinary function calls are no longer reachable. Only deferred functions still execute. `recover()` must be called during this unwinding phase to intercept the panic. A call to `recover()` anywhere else has no effect because there is nothing to intercept.

---

**Q17. What happens if a goroutine panics and there is no `recover()`?**

The entire program crashes. Each goroutine is responsible for its own panic recovery. A panic in goroutine A **cannot** be caught by goroutine B — not even by a goroutine that launched goroutine A.

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered in main:", r) // This does NOT catch panics in other goroutines
        }
    }()

    go func() {
        panic("panic in goroutine") // This crashes the whole program
    }()

    time.Sleep(time.Second)
}
```

---

## Error Handling Patterns

**Q18. What is the early return / guard clause pattern?**

Rather than nesting code in `if err == nil` blocks, return immediately on error:

```go
// Bad — deeply nested
func processOrder(id int) error {
    order, err := db.GetOrder(id)
    if err == nil {
        items, err := db.GetItems(order.ID)
        if err == nil {
            err = sendConfirmation(items)
            if err == nil {
                return nil
            }
            return err
        }
        return err
    }
    return err
}

// Good — early return
func processOrder(id int) error {
    order, err := db.GetOrder(id)
    if err != nil {
        return fmt.Errorf("processOrder: get order %d: %w", id, err)
    }

    items, err := db.GetItems(order.ID)
    if err != nil {
        return fmt.Errorf("processOrder: get items for order %d: %w", id, err)
    }

    if err := sendConfirmation(items); err != nil {
        return fmt.Errorf("processOrder: send confirmation: %w", err)
    }

    return nil
}
```

---

**Q19. What is the multierror pattern and when do you need it?**

When you need to collect multiple errors from independent operations (e.g., validating all fields, processing a batch of items), rather than returning on the first failure:

```go
import "errors"

type MultiError []error

func (m MultiError) Error() string {
    msgs := make([]string, len(m))
    for i, e := range m {
        msgs[i] = e.Error()
    }
    return strings.Join(msgs, "; ")
}

func validateUser(u User) error {
    var errs MultiError
    if u.Name == "" {
        errs = append(errs, errors.New("name is required"))
    }
    if u.Email == "" {
        errs = append(errs, errors.New("email is required"))
    }
    if u.Age < 0 {
        errs = append(errs, errors.New("age cannot be negative"))
    }
    if len(errs) > 0 {
        return errs
    }
    return nil
}
```

Popular third-party implementations: `hashicorp/go-multierror`, `uber-go/multierr`.

---

**Q20. What does Go 1.20+ `errors.Join()` do?**

`errors.Join` combines multiple errors into a single error that `errors.Is` and `errors.As` can traverse:

```go
err1 := errors.New("first failure")
err2 := errors.New("second failure")
combined := errors.Join(err1, err2)

fmt.Println(combined)                        // first failure\nsecond failure
fmt.Println(errors.Is(combined, err1))       // true
fmt.Println(errors.Is(combined, err2))       // true
```

Under the hood it implements `Unwrap() []error` (the multi-unwrap interface introduced in Go 1.20).

---

**Q21. What is the `Unwrap() []error` interface (Go 1.20)?**

Go 1.20 added support for errors that wrap multiple errors simultaneously:

```go
type MultiErr struct {
    Errs []error
}

func (m *MultiErr) Error() string {
    // ... join messages
}

func (m *MultiErr) Unwrap() []error {
    return m.Errs
}
```

`errors.Is` and `errors.As` understand both `Unwrap() error` (single) and `Unwrap() []error` (multiple), traversing the full tree.

---

**Q22. What is the Go convention for error message formatting?**

- Messages should be **lowercase** and have **no trailing punctuation**
- Messages should read as a sequence of context: `"db: query users: scan row: sql: no rows in result set"`
- Each function wraps with its own name as a prefix, creating a breadcrumb trail
- Do not start error messages with "Error:" — the word "error" is implicit

```go
// Bad
return fmt.Errorf("Error opening file: %w.", err)
return fmt.Errorf("Failed to connect to database: %w", err)

// Good
return fmt.Errorf("open config file: %w", err)
return fmt.Errorf("connect to database at %s: %w", addr, err)
```

---

**Q23. What is the `pkg/errors` package and how does it differ from `fmt.Errorf %w`?**

`github.com/pkg/errors` (by Dave Cheney) predates Go 1.13 and provides stack trace capture:

```go
import "github.com/pkg/errors"

func readFile(path string) error {
    _, err := os.Open(path)
    if err != nil {
        return errors.Wrap(err, "readFile")  // captures stack trace HERE
    }
    return nil
}

// Print with stack trace
fmt.Printf("%+v\n", err)
```

`fmt.Errorf("%w", err)` wraps for the purposes of `errors.Is`/`errors.As` but does **not** capture a stack trace. If you need stack traces in production, use `pkg/errors` or a structured logging approach.

---

**Q24. Why is ignoring an error with `_` considered bad practice?**

```go
// Dangerous — silently loses error information
data, _ := os.ReadFile("config.json")

// data will be nil if the file doesn't exist.
// Any subsequent code using data will fail in confusing ways.
```

Go's explicit error handling is only valuable if you actually handle the errors. Ignoring errors is the most common bug pattern in Go codebases. If you genuinely cannot handle an error (e.g., closing a read-only file), at least log it:

```go
if err := f.Close(); err != nil {
    log.Printf("warning: close file: %v", err) // at minimum, log it
}
```

---

**Q25. What is the "don't just check errors, handle them gracefully" principle?**

From Go's error handling philosophy:

1. **Check**: always check the error return
2. **Handle**: do something meaningful — add context, log, translate, return
3. **Don't double-log**: if you log an error and also return it, the caller may log it again

```go
// Anti-pattern: log AND return
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        log.Printf("error reading config: %v", err) // logs here
        return nil, err                              // AND returns — caller may log again
    }
    // ...
}

// Better: just return with context, let the top-level handler log
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("readConfig %s: %w", path, err)
    }
    // ...
}
```

---

**Q26. Can you implement a custom `Is` method on your error type? Why would you?**

Yes. If your error type holds data and two errors of the same type with the same data should be considered equal, implement `Is`:

```go
type HTTPError struct {
    StatusCode int
}

func (e *HTTPError) Error() string {
    return fmt.Sprintf("HTTP error %d", e.StatusCode)
}

func (e *HTTPError) Is(target error) bool {
    t, ok := target.(*HTTPError)
    if !ok {
        return false
    }
    return e.StatusCode == t.StatusCode
}

var ErrNotFound = &HTTPError{StatusCode: 404}

err := fmt.Errorf("handler: %w", &HTTPError{StatusCode: 404})
fmt.Println(errors.Is(err, ErrNotFound)) // true — custom Is compares StatusCode
```

---

**Q27. What is the difference between `fmt.Errorf("%v", err)` and `fmt.Errorf("%w", err)`?**

- `%v` formats the error as a string — the resulting error has **no relationship** to the original; `errors.Is` and `errors.As` cannot find the original.
- `%w` wraps the error — the resulting error implements `Unwrap()` returning the original; `errors.Is` and `errors.As` traverse through it.

```go
original := errors.New("original")

withV := fmt.Errorf("context: %v", original)
withW := fmt.Errorf("context: %w", original)

fmt.Println(errors.Is(withV, original)) // false — string copy only
fmt.Println(errors.Is(withW, original)) // true — wrapped
```

Always use `%w` when you want callers to be able to inspect the underlying error.
