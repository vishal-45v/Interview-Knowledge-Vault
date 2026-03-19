# Chapter 03 — Go Concurrency: Theory Questions

---

## Goroutines

**Q1. What is a goroutine and how does it differ from an OS thread?**

| Property | OS Thread | Goroutine |
|----------|-----------|-----------|
| Managed by | Operating system kernel | Go runtime |
| Initial stack | 1–8 MB (fixed or growable) | ~2 KB (grows/shrinks dynamically) |
| Typical max count | Thousands | Millions |
| Creation cost | ~10–100 µs | ~2–5 µs (100× cheaper) |
| Context switch | Kernel-mode switch (~1 µs) | User-space switch (~200 ns) |
| Scheduling | Preemptive (OS decides) | Cooperative + preemptive (Go runtime) |

```go
// Creating a goroutine — the go keyword:
go func() {
    fmt.Println("I'm running concurrently")
}()

// With a function reference:
go processItem(item)
```

**Q2. How does the Go runtime schedule goroutines? What is the GMP model?**

The Go scheduler uses a GMP (Goroutine, Machine, Processor) model:
- **G (Goroutine)**: A unit of work — a lightweight task with its own stack
- **M (Machine)**: An OS thread — the actual execution unit the kernel sees
- **P (Processor)**: A logical processor — holds a local run queue of goroutines, connects G to M

A P can only be associated with one M at a time. GOMAXPROCS controls how many Ps (and thus OS threads) run concurrently. By default, GOMAXPROCS equals the number of CPU cores.

```
P1 (local queue: G1, G2)  →  M1  →  CPU core 1
P2 (local queue: G3, G4)  →  M2  →  CPU core 2
Global run queue: G5, G6, ...

Work stealing: if P1's queue is empty, it steals from P2 or the global queue
```

**Q3. What is `GOMAXPROCS` and what does it control?**
`GOMAXPROCS` sets the maximum number of OS threads (P's) that can execute user-level Go code simultaneously. It does not limit the total number of goroutines. Set it via:
```go
import "runtime"
runtime.GOMAXPROCS(4)     // use 4 CPU cores
n := runtime.GOMAXPROCS(0) // query current value (0 = query only)
// Also: GOMAXPROCS=4 ./myapp (environment variable)
```
Default: `runtime.NumCPU()` — number of available CPU cores.

**Q4. What is goroutine preemption in Go?**
Early Go versions used cooperative scheduling — goroutines yielded only at certain points (function calls, channel ops, syscalls). Since Go 1.14, goroutines can be preempted at any safe point, even in tight compute loops. This prevents a CPU-bound goroutine from starving others.

```go
// Pre-1.14: this would hog a thread forever on a single-CPU machine
// Post-1.14: the runtime can preempt it
go func() {
    for {
        // no function calls, no blocking ops — tight loop
        // Now preemptible in Go 1.14+
    }
}()
```

---

## Channels

**Q5. What is a channel in Go?**
A channel is a typed conduit for communication between goroutines. Channels are first-class values — you can pass them as function arguments, store them in structs, and close them.

```go
ch := make(chan int)        // unbuffered channel of int
ch := make(chan string, 10) // buffered channel of string, capacity 10

ch <- 42          // send — blocks until receiver is ready (unbuffered)
v := <-ch         // receive — blocks until sender sends

close(ch)         // close the channel — sends the zero value to receivers
v, ok := <-ch     // ok=false means channel is closed and empty
```

**Q6. What is the difference between a buffered and unbuffered channel?**

**Unbuffered channel** (`make(chan T)`):
- Send blocks until there is a receiver
- Receive blocks until there is a sender
- Guarantees synchronous handoff — the send and receive happen together
- Use for: synchronization, guaranteed delivery, rendezvous patterns

**Buffered channel** (`make(chan T, n)`):
- Send only blocks when the buffer is full
- Receive only blocks when the buffer is empty
- Decouples sender and receiver timing
- Use for: work queues, rate limiting, burst absorption

```go
// Unbuffered — synchronous handoff
ch := make(chan int)
go func() { ch <- 42 }()  // blocks until main receives
v := <-ch                  // receives exactly when goroutine sends

// Buffered — asynchronous
ch := make(chan int, 3)
ch <- 1  // doesn't block (buffer has room)
ch <- 2  // doesn't block
ch <- 3  // doesn't block
ch <- 4  // BLOCKS — buffer is full
```

**Q7. What are directional channels in Go?**
Channel types can be restricted to send-only or receive-only. This communicates intent and is enforced by the compiler:

```go
func producer(out chan<- int) { // send-only: can send, cannot receive
    for i := 0; i < 5; i++ {
        out <- i
    }
    close(out)
}

func consumer(in <-chan int) { // receive-only: can receive, cannot send
    for v := range in {
        fmt.Println(v)
    }
}

ch := make(chan int, 5)
go producer(ch) // bidirectional implicitly converts to chan<- int
consumer(ch)    // bidirectional implicitly converts to <-chan int
```

**Q8. What happens when you close a channel?**
- Closed channels can still be read — they drain existing values, then return the zero value with `ok=false`
- Sending to a closed channel causes a **panic**
- Closing a nil channel causes a **panic**
- Closing an already-closed channel causes a **panic**
- Use `close()` to signal no more values will be sent — enables `range` to stop

```go
ch := make(chan int, 3)
ch <- 1; ch <- 2; ch <- 3
close(ch)

for v := range ch {      // range stops when channel is closed and empty
    fmt.Println(v)        // prints 1, 2, 3
}

v, ok := <-ch            // v=0, ok=false — channel closed and empty
```

**Q9. How do you range over a channel?**
```go
ch := make(chan string, 5)
go func() {
    for _, word := range []string{"a", "b", "c"} {
        ch <- word
    }
    close(ch) // MUST close, otherwise range blocks forever
}()

for word := range ch {  // receives values until ch is closed
    fmt.Println(word)
}
```

---

## select Statement

**Q10. What is the `select` statement in Go?**
`select` lets a goroutine wait on multiple channel operations simultaneously. It picks the first case that is ready. If multiple cases are ready, one is chosen at random (not in order).

```go
select {
case v := <-ch1:
    fmt.Println("received from ch1:", v)
case ch2 <- 42:
    fmt.Println("sent to ch2")
case <-time.After(5 * time.Second):
    fmt.Println("timeout")
}
```

**Q11. What is the default case in select?**
A `default` case makes `select` non-blocking. If no other case is ready, `default` runs immediately.

```go
// Non-blocking receive:
select {
case v := <-ch:
    fmt.Println("got:", v)
default:
    fmt.Println("nothing ready")
}

// Non-blocking send:
select {
case ch <- value:
    // sent successfully
default:
    // channel full or no receiver — dropped
}
```

**Q12. How do you implement a timeout with channels?**
```go
// Pattern 1: time.After
func fetchWithTimeout(url string, timeout time.Duration) (string, error) {
    resultCh := make(chan string, 1)
    go func() {
        body, _ := http.Get(url)
        resultCh <- readBody(body)
    }()

    select {
    case result := <-resultCh:
        return result, nil
    case <-time.After(timeout):
        return "", fmt.Errorf("timeout after %v", timeout)
    }
}

// Pattern 2: context.WithTimeout (preferred — cancelable, no goroutine leak)
func fetchWithContext(ctx context.Context, url string) (string, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    body, _ := io.ReadAll(resp.Body)
    return string(body), nil
}
```

---

## sync Package

**Q13. What is `sync.Mutex` and when do you use it?**
`sync.Mutex` provides mutual exclusion — only one goroutine can hold the lock at a time. Use it to protect shared mutable state.

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

**Q14. What is `sync.RWMutex` and when is it preferable to `sync.Mutex`?**
`RWMutex` allows multiple concurrent readers or a single writer. Use it when reads are much more frequent than writes (e.g., a cache).

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock()         // multiple readers can hold RLock simultaneously
    defer c.mu.RUnlock()
    v, ok := c.data[key]
    return v, ok
}

func (c *Cache) Set(key, val string) {
    c.mu.Lock()          // exclusive lock — blocks all readers and writers
    defer c.mu.Unlock()
    c.data[key] = val
}
```

**Q15. What is `sync.WaitGroup` and how do you use it?**
`WaitGroup` waits for a collection of goroutines to finish.

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)             // increment BEFORE launching goroutine
    go func(n int) {
        defer wg.Done()   // decrement when goroutine finishes
        process(n)
    }(i)
}

wg.Wait()                 // block until counter reaches 0
fmt.Println("all done")
```

**Critical**: `wg.Add(1)` must be called in the parent goroutine, before launching the child. Never call `Add` inside the goroutine itself — the parent might reach `Wait` before `Add` is called.

**Q16. What is `sync.Once`?**
`sync.Once` ensures a function is executed exactly once, even if called from multiple goroutines simultaneously. After the function returns, subsequent calls are no-ops.

```go
var (
    once     sync.Once
    instance *Database
)

func GetDB() *Database {
    once.Do(func() {
        instance = connectToDatabase()
    })
    return instance
}
// Safe for concurrent use — connectToDatabase() called exactly once
```

**Q17. What is `sync/atomic` and when do you use it instead of a Mutex?**
The `sync/atomic` package provides atomic operations on primitive types (int32, int64, uint32, uint64, uintptr, unsafe.Pointer). Atomic operations are hardware-level — they execute without locks and are faster than mutex for simple counters and flags.

```go
import "sync/atomic"

var counter int64

// Atomic increment — safe from multiple goroutines
atomic.AddInt64(&counter, 1)

// Atomic load (read) — safe from multiple goroutines
n := atomic.LoadInt64(&counter)

// Atomic compare-and-swap (CAS)
swapped := atomic.CompareAndSwapInt64(&counter, expected, newValue)

// Go 1.19+: typed atomic types
var flag atomic.Bool
flag.Store(true)
fmt.Println(flag.Load()) // true
```

---

## Race Conditions

**Q18. What is a data race in Go?**
A data race occurs when two goroutines access the same memory location concurrently, at least one is a write, and there is no synchronization between them. Data races produce undefined behavior — the program may corrupt data, crash, or produce wrong results.

```go
// DATA RACE — unsynchronized access to count
var count int

go func() { count++ }()  // goroutine 1: reads count, adds 1, writes count
go func() { count++ }()  // goroutine 2: races with goroutine 1

// Use: go run -race main.go  OR  go test -race ./...
// to detect races
```

**Q19. How do you run the Go race detector?**
```bash
go run -race main.go
go test -race ./...
go build -race -o myapp-race ./...
```
The race detector instruments memory accesses and reports any races at runtime. It has ~5-10x overhead — suitable for tests and development, not production.

---

## Context Package

**Q20. What is the `context` package used for?**
The `context` package provides:
1. **Cancellation**: signal to goroutines that work should stop
2. **Deadline/Timeout**: automatic cancellation at a specific time
3. **Value propagation**: pass request-scoped values (trace ID, auth token) through call chains

```go
// context.Background() — the root context, never cancelled
// context.TODO() — placeholder when unsure which context to use

// Cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel() // always cancel to release resources

// Timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Deadline
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// Value
ctx = context.WithValue(ctx, "requestID", "abc-123")
rid := ctx.Value("requestID").(string)
```

**Q21. How do you propagate context through a call chain?**
```go
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // HTTP server already provides a context
    result, err := processRequest(ctx, r)
    // ...
}

func processRequest(ctx context.Context, r *http.Request) (*Result, error) {
    // Pass ctx to every downstream call
    user, err := db.GetUser(ctx, r.UserID)
    if err != nil {
        return nil, err
    }
    return callExternalService(ctx, user)
}

func callExternalService(ctx context.Context, u *User) (*Result, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", serviceURL, nil)
    // If ctx is cancelled (client disconnect, timeout), this request
    // will be cancelled automatically
    resp, err := http.DefaultClient.Do(req)
    // ...
}
```

**Q22. Why should you not store business data in `context.Value`?**
`context.WithValue` uses an untyped key and value (`interface{}`). There is no compile-time type safety, no documentation of what's available, and values may silently not exist. The context package documentation explicitly states context values should only be for request-scoped data that crosses API boundaries (trace IDs, auth tokens), not for passing optional function parameters.

```go
// BAD: using context to pass function parameters
ctx = context.WithValue(ctx, "userID", 42)
ctx = context.WithValue(ctx, "page", 1)
result := getUsers(ctx) // what does getUsers need from ctx? Invisible!

// GOOD: explicit function parameters
result := getUsers(ctx, userID, page) // clear what's needed
```

---

## Concurrency Patterns

**Q23. What is the fan-out pattern in Go?**
Fan-out distributes work from one source to multiple goroutines (workers):

```go
func fanOut(input <-chan Job, numWorkers int) []<-chan Result {
    outputs := make([]<-chan Result, numWorkers)
    for i := 0; i < numWorkers; i++ {
        outputs[i] = worker(input) // each worker reads from same input
    }
    return outputs
}

func worker(input <-chan Job) <-chan Result {
    out := make(chan Result)
    go func() {
        defer close(out)
        for job := range input {
            out <- process(job)
        }
    }()
    return out
}
```

**Q24. What is the fan-in pattern in Go?**
Fan-in merges multiple input channels into one output channel:

```go
func fanIn(channels ...<-chan Result) <-chan Result {
    merged := make(chan Result)
    var wg sync.WaitGroup

    forward := func(ch <-chan Result) {
        defer wg.Done()
        for v := range ch {
            merged <- v
        }
    }

    wg.Add(len(channels))
    for _, ch := range channels {
        go forward(ch)
    }

    go func() {
        wg.Wait()
        close(merged) // close merged when all inputs are done
    }()

    return merged
}
```

**Q25. What is the worker pool pattern?**
A fixed number of goroutines (workers) processes jobs from a shared queue:

```go
func workerPool(numWorkers int, jobs <-chan Job) <-chan Result {
    results := make(chan Result)
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}
```

**Q26. What is the pipeline pattern in Go?**
A pipeline chains stages where each stage receives from the previous and sends to the next:

```go
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func filter(in <-chan int, pred func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if pred(n) {
                out <- n
            }
        }
    }()
    return out
}

// Chain the pipeline:
nums := generate(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
squared := square(nums)
evens := filter(squared, func(n int) bool { return n%2 == 0 })

for v := range evens {
    fmt.Println(v) // 4, 16, 36, 64, 100
}
```

---

## Go Memory Model

**Q27. What is the Go memory model?**
The Go memory model specifies the conditions under which reads of a variable in one goroutine are guaranteed to observe the value written by a write in another goroutine. The key concept is "happens-before": if event A happens before event B, B is guaranteed to see the effects of A.

Happens-before guarantees are established by:
- Channel operations (send happens before receive)
- `sync.Mutex` lock/unlock
- `sync.Once.Do` completion
- Goroutine creation (`go` statement happens before the goroutine body starts)
- `sync.WaitGroup.Done` happens before `Wait` returns

**Q28. Can you have a data race with channel operations?**
Channel operations are synchronization points — they establish happens-before relationships and are inherently race-free. However, the data accessed *before* sending or *after* receiving can still have races if shared without synchronization.

```go
// Safe: channel establishes happens-before
data := compute()
ch <- data       // send
// Guarantee: receiver sees the computed data

v := <-ch        // receive
use(v)           // Safe — happens after send

// RACE: sharing data without synchronization around channel ops
var shared int
go func() {
    shared = 42  // write
    ch <- 1      // signal
}()
<-ch             // receive
fmt.Println(shared) // Safe — channel establishes happens-before
// But reading shared WITHOUT the channel would be a race
```

---

## errgroup

**Q29. What is `errgroup` and when do you use it?**
`errgroup` (from `golang.org/x/sync/errgroup`) provides goroutine management with error collection — a cleaner alternative to `WaitGroup` + error channels when running multiple fallible concurrent operations.

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]string, len(urls))

    for i, url := range urls {
        i, url := i, url // capture loop variables
        g.Go(func() error {
            body, err := fetchURL(ctx, url)
            if err != nil {
                return fmt.Errorf("fetching %s: %w", url, err)
            }
            results[i] = body
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err // returns the first non-nil error
    }
    return results, nil
}
```

---

## Goroutine Leaks

**Q30. What is a goroutine leak?**
A goroutine leak occurs when a goroutine is started but never terminates — it remains blocked indefinitely, consuming memory and potentially a thread. Common causes:
- Sending to a channel with no receiver
- Receiving from a channel that's never sent to and never closed
- Blocking on a select with no matching case and no default
- Infinite loop with no exit condition

```go
// LEAK: goroutine blocked forever waiting to send
func leak() {
    ch := make(chan int)
    go func() {
        ch <- 42 // blocks forever if nobody reads from ch
    }()
    // function returns, ch goes out of scope, but goroutine is stuck
}

// Detection: goleak package, runtime.NumGoroutine()
// Prevention: always provide a way for goroutines to exit (done channel, context)
```

**Q31. How do you prevent goroutine leaks?**
```go
// Pattern 1: done channel
func work(done <-chan struct{}) {
    for {
        select {
        case <-done:
            return // exit when signalled
        default:
            doWork()
        }
    }
}

done := make(chan struct{})
go work(done)
// Later:
close(done) // signals all goroutines reading <-done

// Pattern 2: context (preferred)
func work(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            doWork()
        }
    }
}

ctx, cancel := context.WithCancel(context.Background())
go work(ctx)
defer cancel() // signal goroutine to stop
```

---

## Channels — Additional

**Q32. What happens when you send to or receive from a nil channel?**
- Sending to a nil channel blocks forever
- Receiving from a nil channel blocks forever
- Closing a nil channel panics

```go
var ch chan int // nil channel

go func() { ch <- 1 }() // goroutine blocks forever — LEAK
v := <-ch               // blocks forever
close(ch)               // PANIC: close of nil channel
```

Nil channel in `select` is useful: it effectively disables a case.

**Q33. How do you use a nil channel as a disable trick in select?**
```go
func merge(a, b <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for a != nil || b != nil {
            select {
            case v, ok := <-a:
                if !ok {
                    a = nil // disable this case once channel is closed
                    continue
                }
                out <- v
            case v, ok := <-b:
                if !ok {
                    b = nil // disable this case
                    continue
                }
                out <- v
            }
        }
    }()
    return out
}
```

**Q34. What is `sync.Cond` and when would you use it?**
`sync.Cond` implements a condition variable — it allows goroutines to wait for a condition to become true, with efficient signaling (no busy-waiting).

```go
var mu sync.Mutex
var cond = sync.NewCond(&mu)
var ready bool

// Producer: set condition and signal
go func() {
    mu.Lock()
    ready = true
    cond.Signal()  // wake one waiting goroutine
    // cond.Broadcast() // wake all waiting goroutines
    mu.Unlock()
}()

// Consumer: wait for condition
mu.Lock()
for !ready {  // must loop to handle spurious wakeups
    cond.Wait() // releases mu, waits, re-acquires mu on wake
}
// condition is now true
mu.Unlock()
```

**Q35. What is the difference between `sync.Cond.Signal` and `sync.Cond.Broadcast`?**
- `Signal()` wakes exactly one goroutine waiting on the condition
- `Broadcast()` wakes all goroutines waiting on the condition
- Use `Signal` when only one goroutine can proceed (e.g., one item in a queue)
- Use `Broadcast` when all waiters should re-check (e.g., a configuration change)

---

**Total: 35 theory questions covering goroutines, channels, select, sync package, context, GMP model, patterns, and memory model.**
