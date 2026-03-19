# Chapter 03 — Go Concurrency: Follow-Up Traps

---

## Trap 1: Goroutine leak — channel send with no receiver

```go
// LEAK: goroutine is blocked forever
func leaky() <-chan int {
    ch := make(chan int) // unbuffered
    go func() {
        result := compute() // returns eventually
        ch <- result         // BLOCKS if caller discards the returned channel
    }()
    return ch
}

func main() {
    _ = leaky() // discard the channel — goroutine is now stuck forever
    // runtime.NumGoroutine() increases by 1 permanently
}

// DETECTION:
// 1. goleak package in tests
import "go.uber.org/goleak"
func TestNoLeaks(t *testing.T) {
    defer goleak.VerifyNone(t)
    // ... test code
}

// 2. runtime.NumGoroutine() before and after
before := runtime.NumGoroutine()
leaky()
runtime.GC()
time.Sleep(time.Millisecond)
after := runtime.NumGoroutine()
if after > before {
    // goroutine leaked
}

// FIX: use buffered channel so goroutine doesn't block
func fixed() <-chan int {
    ch := make(chan int, 1) // goroutine can send and exit
    go func() {
        ch <- compute()
    }()
    return ch
}
```

---

## Trap 2: Closing a nil channel panics

```go
var ch chan int // nil channel

close(ch) // PANIC: close of nil channel

// Common scenario: conditional channel creation
func maybeMakeChannel(create bool) chan int {
    var ch chan int
    if create {
        ch = make(chan int)
    }
    return ch
}

ch := maybeMakeChannel(false)
// close(ch) — would panic!

// Safe close helper:
func safeCClose[T any](ch chan T) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("attempted to close nil or already-closed channel: %v", r)
        }
    }()
    close(ch)
}

// Better: use sync.Once for one-time close
type SafeChannel[T any] struct {
    ch   chan T
    once sync.Once
}

func (sc *SafeChannel[T]) Close() {
    sc.once.Do(func() { close(sc.ch) })
}
```

---

## Trap 3: Sending on a closed channel panics

```go
ch := make(chan int, 3)
ch <- 1
ch <- 2
close(ch)

ch <- 3 // PANIC: send on closed channel

// The receiver can tell if a channel is closed:
v, ok := <-ch // ok=false means closed
// But the SENDER has no way to check if a channel is closed safely

// RULE: Only the SENDER should close a channel.
// If multiple senders exist, use a sync.Once or a dedicated "done" signal.

// Pattern for multiple senders:
var wg sync.WaitGroup
out := make(chan int, 10)

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        out <- n * n
    }(i)
}

// A separate goroutine closes the channel when all senders are done
go func() {
    wg.Wait()
    close(out) // safe: exactly one close
}()

for v := range out {
    fmt.Println(v)
}
```

---

## Trap 4: Range over channel blocks forever if channel not closed

```go
ch := make(chan int, 5)
for i := 0; i < 5; i++ {
    ch <- i
}
// Forgot to close(ch)!

for v := range ch { // BLOCKS after consuming the 5 values — never exits
    fmt.Println(v)
}
// The range only terminates when ch is closed

// Always close channels that are being ranged over.
// The responsibility lies with the SENDER / PRODUCER.

// Common test issue:
func TestWorker(t *testing.T) {
    jobs := make(chan int, 10)
    results := make(chan int, 10)

    go worker(jobs, results)

    jobs <- 1
    jobs <- 2
    // close(jobs) ← forgot this!

    for r := range results { // blocks forever — worker is waiting for more jobs
        t.Log(r)
    }
}
```

---

## Trap 5: Loop variable capture in goroutines (the classic pre-1.22 trap)

```go
// Pre-Go 1.22: loop variable is ONE variable shared across all iterations
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(i) // captures the variable i, not its current value
    }()
}
wg.Wait()
// Likely prints: 5 5 5 5 5 (all see the final value of i)
// Sometimes: 3 5 5 5 5 (race conditions make it non-deterministic)

// Fix 1: Shadow the variable (works in all Go versions)
for i := 0; i < 5; i++ {
    i := i // new variable per iteration
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(i)
    }()
}

// Fix 2: Pass as parameter (explicit and clear)
for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(n int) { // n is a parameter — its own copy
        defer wg.Done()
        fmt.Println(n)
    }(i)
}

// Go 1.22+: loop variables are per-iteration — no fix needed
// Each iteration creates a new i — closures work correctly
```

This also applies to `range` loops:
```go
// Pre-1.22:
for _, item := range items {
    item := item // fix needed
    go process(item)
}
```

---

## Trap 6: WaitGroup.Add() must be called before goroutine starts

```go
// RACE: Add() called inside goroutine — may not run before Wait()
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
    go func() {
        wg.Add(1) // WRONG: might run after wg.Wait()
        defer wg.Done()
        doWork()
    }()
}
wg.Wait() // may return before all goroutines start!

// CORRECT: Add() in the parent, before launching
for i := 0; i < 5; i++ {
    wg.Add(1) // increment before go statement
    go func() {
        defer wg.Done()
        doWork()
    }()
}
wg.Wait() // guaranteed to wait for all goroutines

// The go statement schedules the goroutine but doesn't guarantee
// when it starts running. Add() must execute before Wait() can return.
```

---

## Trap 7: sync.Mutex is not reentrant (no recursive locking)

```go
type Cache struct {
    mu   sync.Mutex
    data map[string]int
}

func (c *Cache) Get(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.data[key]
}

func (c *Cache) GetOrSet(key string, defaultVal int) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    // ...
    val := c.Get(key) // DEADLOCK: tries to acquire mu again — not reentrant!
    // ...
    return val
}

// Fix 1: Use an internal unlocked helper
func (c *Cache) get(key string) int {  // unlocked version
    return c.data[key]
}

func (c *Cache) Get(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.get(key)
}

func (c *Cache) GetOrSet(key string, defaultVal int) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    val := c.get(key) // calls unlocked helper — no deadlock
    if val == 0 {
        c.data[key] = defaultVal
        return defaultVal
    }
    return val
}
```

---

## Trap 8: select with a nil channel case blocks on that case (useful trick)

```go
// A nil channel case in select is NEVER selected
// This is actually a useful feature for disabling cases dynamically

func merge(a, b <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for a != nil || b != nil {
            select {
            case v, ok := <-a:
                if !ok {
                    a = nil  // disable this case — nil channel never selected
                    continue
                }
                out <- v
            case v, ok := <-b:
                if !ok {
                    b = nil  // disable this case
                    continue
                }
                out <- v
            }
        }
    }()
    return out
}

// Demonstration of nil channel in select:
var ch chan int = nil
select {
case v := <-ch: // nil channel — this case is NEVER selected
    fmt.Println(v)
case <-time.After(time.Millisecond):
    fmt.Println("timeout") // this fires
}
```

---

## Trap 9: Context.Value — don't use it for business data

```go
// BAD: using context for function parameters
type ctxKey string

func getUsers(ctx context.Context) ([]User, error) {
    limit, ok := ctx.Value(ctxKey("limit")).(int) // no compile-time safety
    if !ok {
        limit = 10 // silently uses default — hard to debug
    }
    // ...
}

// Called with context carrying the limit:
ctx = context.WithValue(ctx, ctxKey("limit"), 25)
users, _ := getUsers(ctx)

// Problems:
// 1. Not visible in function signature — hidden dependency
// 2. No compile-time type checking
// 3. Caller might forget to set the value — function silently uses default
// 4. Value key collisions between packages

// GOOD: explicit parameter
func getUsers(ctx context.Context, limit int) ([]User, error) {
    // ...
}
users, _ := getUsers(ctx, 25) // clear what's being passed

// ACCEPTABLE context values: request-scoped, cross-cutting concerns
// - Request ID / Trace ID (for logging)
// - Auth token / User session (set once, read in middleware and handlers)
// - Deadline / Cancellation (built into context)
type requestIDKey struct{}

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey{}, id)
}

func GetRequestID(ctx context.Context) string {
    id, _ := ctx.Value(requestIDKey{}).(string)
    return id
}
```

---

## Trap 10: sync.Once and panic — function is permanently "done"

```go
var once sync.Once
var db *sql.DB

func InitDB() error {
    var err error
    once.Do(func() {
        db, err = sql.Open("postgres", dsn)
        if err != nil {
            return // err is set, but once marks it as done!
        }
        err = db.Ping()
        // If Ping panics, once is permanently done, db might be non-nil but broken
    })
    return err
}

// BUG: if InitDB fails on first call, subsequent calls are no-ops
// and always return nil error (once.Do doesn't re-run)!

err1 := InitDB() // say this fails — db is nil, err1 is non-nil
err2 := InitDB() // once.Do is a no-op — err2 is nil even though db is still nil!
// Callers after err1 think initialization succeeded!

// FIX: store the error as a package-level variable
var (
    once  sync.Once
    db    *sql.DB
    dbErr error
)

func GetDB() (*sql.DB, error) {
    once.Do(func() {
        db, dbErr = sql.Open("postgres", dsn)
        if dbErr == nil {
            dbErr = db.Ping()
        }
    })
    return db, dbErr // always returns the result of the one-time initialization
}
```

---

## Trap 11: Mutex lock copied by value

```go
type Counter struct {
    sync.Mutex
    count int
}

// TRAP: passing Counter by value copies the Mutex
func badUsage(c Counter) { // copies the Mutex!
    c.Lock()
    defer c.Unlock()
    c.count++ // modifies the copy, not the original
}

// go vet catches this:
// "passes lock by value: Counter contains sync.Mutex"

// FIX: always pass Mutex-containing types by pointer
func goodUsage(c *Counter) {
    c.Lock()
    defer c.Unlock()
    c.count++
}

// Similarly, never copy a WaitGroup, Cond, or any sync type
type Service struct {
    wg sync.WaitGroup  // embedded — NEVER copy Service by value after use
}

func processService(s Service) { // WRONG: copies WaitGroup
}

func processService(s *Service) { // CORRECT
}
```

---

## Trap 12: time.After leaks a timer goroutine until it fires

```go
// LEAK: time.After creates a channel that is kept alive by the runtime timer
// until it fires — even if you stopped using the channel earlier
func processWithTimeout(ch <-chan Data, timeout time.Duration) {
    for {
        select {
        case data := <-ch:
            handleData(data)
        case <-time.After(timeout): // NEW TIMER EVERY ITERATION!
            log.Println("timeout waiting for data")
            return
        }
    }
}
// If timeout is 30 seconds and we're processing fast,
// we create many timer goroutines that live for 30 seconds each

// FIX: create the timer ONCE outside the loop
func processWithTimeoutFixed(ch <-chan Data, timeout time.Duration) {
    timer := time.NewTimer(timeout)
    defer timer.Stop() // reclaim resources immediately

    for {
        // Reset the timer for each iteration if you want per-operation timeout
        timer.Reset(timeout)

        select {
        case data := <-ch:
            handleData(data)
        case <-timer.C:
            log.Println("timeout")
            return
        }
    }
}
```
