# Go Error Handling — Follow-Up Traps

## Trap 1: Using `==` Instead of `errors.Is` for Wrapped Errors

**The trap:** Comparing errors with `==` fails when the error has been wrapped.

```go
var ErrNotFound = errors.New("not found")

func getUser(id int) error {
    return fmt.Errorf("db.GetUser: %w", ErrNotFound) // wrapped with context
}

err := getUser(42)

// WRONG — returns false because err != ErrNotFound (it's a *fmt.wrapError)
if err == ErrNotFound {
    fmt.Println("not found")
}

// CORRECT — traverses the chain, finds ErrNotFound
if errors.Is(err, ErrNotFound) {
    fmt.Println("not found")
}
```

**Rule:** Never use `==` to compare errors that may have been wrapped. Always use `errors.Is`.

---

## Trap 2: `errors.As` Requires a Pointer to a Pointer

**The trap:** Passing the wrong type to `errors.As`.

```go
type MyError struct{ Code int }
func (e *MyError) Error() string { return fmt.Sprintf("code %d", e.Code) }

err := fmt.Errorf("wrap: %w", &MyError{Code: 42})

// WRONG — passing *MyError, not **MyError
var target *MyError
if errors.As(err, target) { // panics or doesn't work
    fmt.Println(target.Code)
}

// CORRECT — pass pointer-to-pointer so errors.As can set it
var target2 *MyError
if errors.As(err, &target2) {
    fmt.Println(target2.Code) // 42
}
```

`errors.As` needs `&target` so it can assign the found error value to your variable.

---

## Trap 3: The Non-Nil Nil Interface Trap

**The trap:** Returning a typed nil pointer as an `error` interface produces a non-nil interface.

```go
type MyError struct{ Code int }
func (e *MyError) Error() string { return fmt.Sprintf("code %d", e.Code) }

func mayFail(fail bool) error {
    var err *MyError // typed nil — *MyError(nil)
    if fail {
        err = &MyError{Code: 500}
    }
    return err // BUG: returns non-nil interface even when err is nil
}

err := mayFail(false)
if err != nil {
    fmt.Println("ERROR:", err) // This prints! err is non-nil interface wrapping nil pointer
}
```

**Why it happens:** An interface value has two fields: (type, value). `(*MyError)(nil)` has type `*MyError` and value `nil` — the interface itself is not nil.

**Fix:** Return the untyped `nil`:

```go
func mayFail(fail bool) error {
    if fail {
        return &MyError{Code: 500}
    }
    return nil // untyped nil — interface is truly nil
}
```

**General rule:** Never return a concrete typed nil as an `error`. Return the bare `nil`.

---

## Trap 4: Panic in a Goroutine Cannot Be Recovered by Another Goroutine

**The trap:** Thinking a `recover()` in one goroutine will catch panics from a different goroutine.

```go
func main() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("recovered in main:", r) // This NEVER catches goroutine panics
        }
    }()

    go func() {
        panic("goroutine panic") // This crashes the ENTIRE program
    }()

    time.Sleep(time.Second)
    fmt.Println("main continues") // Never reached
}
```

**Fix:** Each goroutine must have its own recovery:

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("goroutine panicked: %v", r)
        }
    }()
    panic("goroutine panic") // now recovered
}()
```

---

## Trap 5: `fmt.Errorf("%v", err)` Does Not Wrap — Use `%w`

**The trap:** Using `%v` when you intend to wrap an error for `errors.Is`/`errors.As`.

```go
original := errors.New("database connection refused")

// %v — creates a new error with the string representation, loses the original
withV := fmt.Errorf("connect: %v", original)

// %w — wraps the original, preserves the chain
withW := fmt.Errorf("connect: %w", original)

fmt.Println(errors.Is(withV, original)) // false — NOT wrapped
fmt.Println(errors.Is(withW, original)) // true — properly wrapped
```

When to use each:
- `%w` — when you want callers to be able to `errors.Is` or `errors.As` through your error
- `%v` — when you are creating a completely new error and don't want callers to inspect the original (rare)

---

## Trap 6: `defer` + `recover()` Must Be in the Same Goroutine

**The trap:** Trying to recover from a panic across goroutine boundaries via deferred functions.

```go
func badRecover() {
    defer func() {
        r := recover()
        fmt.Println("recovered:", r) // Never called for goroutine below
    }()

    go func() {
        panic("boom") // crashes program — different goroutine
    }()

    time.Sleep(time.Second)
}
```

Also: `recover()` only works when called **directly** inside a deferred function — not in a function called by a deferred function:

```go
func helper() {
    r := recover() // This does NOT work — not directly in deferred func
    fmt.Println("recovered:", r)
}

func bad() {
    defer helper() // recover() in helper() has no effect
    panic("boom")  // program crashes
}

func good() {
    defer func() {
        r := recover() // This works — directly in deferred func
        fmt.Println("recovered:", r)
    }()
    panic("boom") // recovered
}
```

---

## Trap 7: Sentinel Error Equality Depends on Pointer Identity

**The trap:** Recreating sentinel errors with `errors.New` in different packages gives different pointers that are not equal.

```go
// package a
var ErrTimeout = errors.New("timeout")

// package b — accidentally redefines the same-message error
var ErrTimeout = errors.New("timeout")

// In caller:
err := a.ErrTimeout
fmt.Println(errors.Is(err, b.ErrTimeout)) // false — different pointers!
```

`errors.New` creates a new `*errorString` each time it is called. Two calls with the same string produce two different pointers.

**Fix:** Define sentinel errors in one authoritative location and import them:

```go
import "myapp/errs"

// Everyone uses errs.ErrTimeout
if errors.Is(err, errs.ErrTimeout) { ... }
```

---

## Trap 8: Swallowing Errors with `_`

**The trap:** The most common Go bug — silently discarding errors.

```go
// All of these silently discard errors:
data, _ := os.ReadFile("config.json") // data is nil if file missing
conn, _ := db.Connect(dsn)            // conn is nil if connection fails
_ = f.Close()                          // close error lost forever

// What happens next is mysterious and hard to debug
json.Unmarshal(data, &cfg)            // panics or silently produces zero values
conn.Query("SELECT ...")               // nil pointer dereference
```

**Rule:** Treat every error as important until proven otherwise. At minimum, log it:

```go
if err := f.Close(); err != nil {
    log.Printf("close file: %v", err)
}
```

---

## Trap 9: Error Message Convention Violations

**The trap:** Starting error messages with capital letters, ending with periods, or prefixing with "Error:".

```go
// BAD — these are all wrong
return errors.New("Error: file not found.")
return fmt.Errorf("Failed to connect to database at %s", addr)
return errors.New("Cannot parse JSON.")

// GOOD — lowercase, no trailing punctuation
return errors.New("file not found")
return fmt.Errorf("connect to database at %s: %w", addr, err)
return errors.New("cannot parse JSON")
```

**Why:** Error messages are often composed into longer strings by callers:
`"processRequest: connect to database at localhost:5432: dial tcp: connection refused"`

If each part starts with a capital letter or ends with a period, the composed message looks wrong.

---

## Trap 10: Returning Both a Non-nil Error and Modified Output Parameters

**The trap:** When a function returns an error, should the caller trust any other return values?

```go
func processItems(items []Item) ([]Result, error) {
    var results []Result
    for _, item := range items {
        r, err := process(item)
        if err != nil {
            return results, err // Partial results AND error — confusing!
        }
        results = append(results, r)
    }
    return results, nil
}

// Caller doesn't know if partial results are valid
results, err := processItems(items)
if err != nil {
    // Can I use results? It depends on the function's contract — not obvious!
}
```

**Rule:** Establish a clear contract. Either:
1. **Return nil for output on error**: callers never inspect output when error is non-nil
2. **Document that partial results are valid**: explicitly state in the doc comment

The Go standard library generally returns nil/zero values when returning an error.

```go
// Clear contract: on error, results is nil
func processItems(items []Item) ([]Result, error) {
    var results []Result
    for _, item := range items {
        r, err := process(item)
        if err != nil {
            return nil, fmt.Errorf("process item %v: %w", item, err) // nil results
        }
        results = append(results, r)
    }
    return results, nil
}
```

---

## Trap 11: Wrapping Errors Multiple Times in the Same Function

**The trap:** Wrapping an error more than once in the same function call chain, creating redundant context.

```go
func getUser(id int) (*User, error) {
    user, err := db.QueryUser(id)
    if err != nil {
        // Wrapping once — correct
        return nil, fmt.Errorf("getUser %d: %w", id, err)
    }
    return user, nil
}

func handleRequest(id int) error {
    user, err := getUser(id)
    if err != nil {
        // DON'T wrap again with same context — creates noise
        return fmt.Errorf("getUser %d: %w", id, err) // duplicates "getUser 42"
    }
    _ = user
    return nil
}
// Error: "getUser 42: getUser 42: sql: no rows"
```

**Fix:** Each layer wraps with **its own** context, not the callee's:

```go
func handleRequest(id int) error {
    user, err := getUser(id)
    if err != nil {
        return fmt.Errorf("handleRequest: %w", err) // adds THIS layer's context
    }
    _ = user
    return nil
}
// Error: "handleRequest: getUser 42: sql: no rows"
```

---

## Trap 12: Using `panic` for Expected Error Conditions

**The trap:** Using `panic` as a shortcut to avoid returning errors.

```go
// WRONG — config missing is an expected condition in production
func loadConfig(path string) *Config {
    data, err := os.ReadFile(path)
    if err != nil {
        panic(err) // Crashes the server on missing config file!
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        panic(err)
    }
    return &cfg
}

// CORRECT — return the error, let the caller decide
func loadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("load config from %s: %w", path, err)
    }
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        return nil, fmt.Errorf("parse config from %s: %w", path, err)
    }
    return &cfg, nil
}
```

Panic is appropriate only for: programming errors (invariant violations), `Must*` wrapper functions, and truly unrecoverable initialization conditions.
