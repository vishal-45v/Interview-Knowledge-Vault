# Chapter 03 — Go Concurrency: Structured Answers

---

## Answer 1: How is a goroutine different from an OS thread?

**OS Thread:**
- Created and scheduled by the operating system kernel
- Fixed stack size: typically 1–8 MB per thread
- Context switch requires a kernel mode switch (~1–10 µs)
- Creating a thread is expensive (memory allocation, kernel state)
- Maximum practical count: thousands on a typical system

**Goroutine:**
- Created and scheduled by the Go runtime (user-space scheduler)
- Dynamic stack: starts at ~2 KB, grows and shrinks as needed (up to 1 GB by default)
- Context switch is in user space (~200 ns)
- Creating a goroutine is cheap (~2–5 µs, ~2 KB stack allocation)
- Maximum practical count: hundreds of thousands to millions

```go
// Demonstrating goroutine creation scale
func main() {
    var wg sync.WaitGroup

    // Spawning 1,000,000 goroutines — feasible in Go
    // 1,000,000 OS threads would be impossible (too much memory)
    for i := 0; i < 1_000_000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            time.Sleep(time.Second)
        }()
    }
    wg.Wait()
    // Uses roughly 2 GB memory (1M × 2KB initial stack)
    // Compared to 1M threads × 8MB = 8 TB for OS threads
}
```

---

## Answer 2: What is the Go scheduler (GMP model)?

The Go scheduler uses a three-layer model: **G**oroutines, **M**achine threads, and **P**rocessors.

```
Goroutines (G): units of work
    G1  G2  G3  G4  G5  G6   ← goroutine queue

Processors (P): local run queues + scheduler state
    P1[G1, G2]    P2[G3, G4]   ← each P has a local queue

Machine threads (M): OS threads executing P's goroutines
    M1 runs P1    M2 runs P2   ← GOMAXPROCS determines max active P/M pairs

Global queue: [G5, G6]  ← overflow when P queues are full
```

**Scheduling mechanics:**
1. When a goroutine is created with `go`, it's placed in the creating P's local run queue
2. If the local queue is full (>256), half are moved to the global queue
3. An M picks a goroutine from its P's local queue and executes it
4. **Work stealing**: when a P's queue is empty, it steals goroutines from other Ps' queues (random half)
5. When a goroutine blocks on a syscall, the M is detached from P, a new M takes over P, and the blocking goroutine waits for the syscall to complete

**Preemption:**
- Since Go 1.14, goroutines can be preempted at any safe point (not just function calls)
- Prevents CPU-bound goroutines from starving others
- Uses signals (SIGURG) to interrupt running goroutines at safe points

```go
// GOMAXPROCS controls parallelism
runtime.GOMAXPROCS(runtime.NumCPU()) // default: use all CPU cores
runtime.GOMAXPROCS(1)  // single-threaded execution (useful for debugging races)
```

---

## Answer 3: When should you use a buffered vs unbuffered channel?

**Unbuffered channel** (`make(chan T)`): Synchronous handoff. The sender blocks until a receiver is ready, and the receiver blocks until a sender sends. Use when:
- You need guaranteed delivery — the message was definitely received
- You need to synchronize two goroutines at a point
- You're implementing rendezvous patterns

**Buffered channel** (`make(chan T, n)`): Asynchronous with a limited buffer. Sender only blocks when the buffer is full, receiver only blocks when empty. Use when:
- You want to decouple producer and receiver timing
- You know the maximum number of items to be sent (bounded work queue)
- You're implementing a semaphore (capacity = max concurrent)

```go
// Unbuffered: rendezvous / synchronization
done := make(chan struct{})
go func() {
    doWork()
    close(done) // signal completion
}()
<-done // wait for goroutine to finish

// Buffered: work queue with bounded capacity
jobs := make(chan Job, 100) // accepts 100 jobs before blocking producer

// Buffered: semaphore (limit concurrent DB connections)
sem := make(chan struct{}, 10) // max 10 concurrent
for _, item := range items {
    sem <- struct{}{}         // acquire
    go func() {
        defer func() { <-sem }() // release
        processWithDB(item)
    }()
}

// Buffered: fire-and-forget result collection
results := make(chan Result, len(jobs))
for _, j := range jobs {
    go func(job Job) {
        results <- process(job) // never blocks — results has enough space
    }(j)
}
for range jobs {
    r := <-results
    _ = r
}
```

**Key rule:** Buffered channels do not eliminate synchronization problems — they just reduce contention at the cost of latency. Don't use large buffers to hide race conditions.

---

## Answer 4: Implement a worker pool pattern in Go

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID      int
    Payload string
}

type Result struct {
    JobID    int
    Output   string
    Duration time.Duration
    Err      error
}

// WorkerPool processes jobs from a channel using N workers.
// The returned channel is closed when all jobs have been processed.
func WorkerPool(ctx context.Context, numWorkers int, jobs <-chan Job) <-chan Result {
    results := make(chan Result, numWorkers*2) // buffer to avoid blocking workers
    var wg sync.WaitGroup

    worker := func(id int) {
        defer wg.Done()
        for {
            select {
            case job, ok := <-jobs:
                if !ok {
                    return // channel closed, no more jobs
                }
                start := time.Now()
                output, err := processJob(ctx, job)
                results <- Result{
                    JobID:    job.ID,
                    Output:   output,
                    Duration: time.Since(start),
                    Err:      err,
                }
            case <-ctx.Done():
                return // context cancelled
            }
        }
    }

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go worker(i)
    }

    // Close results when all workers finish
    go func() {
        wg.Wait()
        close(results)
    }()

    return results
}

func processJob(ctx context.Context, j Job) (string, error) {
    select {
    case <-time.After(50 * time.Millisecond): // simulate work
        return fmt.Sprintf("processed: %s", j.Payload), nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Feed jobs
    jobs := make(chan Job, 50)
    go func() {
        defer close(jobs)
        for i := 0; i < 20; i++ {
            select {
            case jobs <- Job{ID: i, Payload: fmt.Sprintf("data-%d", i)}:
            case <-ctx.Done():
                return
            }
        }
    }()

    // Process with 4 workers
    results := WorkerPool(ctx, 4, jobs)

    // Collect results
    var processed, failed int
    for r := range results {
        if r.Err != nil {
            failed++
            fmt.Printf("job %d FAILED: %v\n", r.JobID, r.Err)
        } else {
            processed++
            fmt.Printf("job %d OK [%v]: %s\n", r.JobID, r.Duration, r.Output)
        }
    }
    fmt.Printf("processed: %d, failed: %d\n", processed, failed)
}
```

---

## Answer 5: How does context cancellation work? Implement a function with timeout.

Context forms a tree. When a parent context is cancelled (or times out), all its children are cancelled automatically. This allows a single cancellation signal to propagate through an entire request-handling chain.

```go
func processWithTimeout(data []string) ([]string, error) {
    // Create a context that auto-cancels after 3 seconds
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel() // Always defer cancel — releases resources even if timeout isn't hit

    results := make([]string, 0, len(data))

    for _, item := range data {
        // Check if context is already done before starting expensive work
        select {
        case <-ctx.Done():
            return results, fmt.Errorf("processing cancelled: %w", ctx.Err())
        default:
        }

        result, err := processItemWithCtx(ctx, item)
        if err != nil {
            if errors.Is(err, context.DeadlineExceeded) {
                return results, fmt.Errorf("deadline exceeded after processing %d items", len(results))
            }
            return results, err
        }
        results = append(results, result)
    }
    return results, nil
}

func processItemWithCtx(ctx context.Context, item string) (string, error) {
    // Simulate an async operation that respects cancellation
    done := make(chan string, 1)
    go func() {
        done <- expensiveTransform(item) // simulate work
    }()

    select {
    case result := <-done:
        return result, nil
    case <-ctx.Done():
        return "", ctx.Err() // DeadlineExceeded or Canceled
    }
}

// Context tree propagation:
func handler(ctx context.Context) {
    // Create a child context with additional timeout
    childCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    // Pass childCtx to downstream calls
    // If the parent ctx is cancelled (e.g., HTTP client disconnect),
    // childCtx is also cancelled automatically
    result, err := downstreamService(childCtx)
    if errors.Is(err, context.Canceled) {
        log.Println("client disconnected")
        return
    }
    _ = result
}
```

---

## Answer 6: What is a goroutine leak and how do you detect/prevent it?

A goroutine leak is a goroutine that was created but never terminates — it stays alive indefinitely, holding its stack memory and possibly other resources.

**Common causes:**

```go
// Cause 1: Channel with no receiver
func leak1() {
    ch := make(chan int)
    go func() {
        ch <- 1 // blocks forever — nobody reads
    }()
}

// Cause 2: Receive from channel that's never closed
func leak2() {
    ch := make(chan int)
    go func() {
        for v := range ch { // blocks forever — ch never closed
            _ = v
        }
    }()
}

// Cause 3: Select with no matching case and no timeout/context
func leak3() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    go func() {
        select {
        case v := <-ch1: _ = v
        case v := <-ch2: _ = v
        // No default, no timeout — goroutine waits indefinitely
        }
    }()
}
```

**Detection:**

```go
// 1. runtime.NumGoroutine() — simple count
before := runtime.NumGoroutine()
runCode()
runtime.GC()
time.Sleep(10 * time.Millisecond) // let goroutines finish
after := runtime.NumGoroutine()
if after > before {
    panic(fmt.Sprintf("goroutine leak: %d > %d", after, before))
}

// 2. goleak in tests (https://github.com/uber-go/goleak)
func TestMyCode(t *testing.T) {
    defer goleak.VerifyNone(t)
    runCode() // any leaked goroutines are reported
}

// 3. pprof goroutine profile
go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
// GET http://localhost:6060/debug/pprof/goroutine?debug=2
// Shows all goroutine stacks
```

**Prevention:**

```go
// Always provide an exit path:
func noLeak(ctx context.Context) <-chan int {
    out := make(chan int, 1) // buffered: goroutine can always send
    go func() {
        defer close(out)
        select {
        case out <- compute():
        case <-ctx.Done(): // exit if cancelled
        }
    }()
    return out
}
```

---

## Answer 7: What is the difference between sync.Mutex and sync.RWMutex?

```go
// sync.Mutex: exclusive lock
// - Only ONE goroutine at a time (reader OR writer)
// - Use when reads and writes are equal or writes dominate

type ExclusiveCache struct {
    mu   sync.Mutex
    data map[string]int
}

func (c *ExclusiveCache) Get(key string) int {
    c.mu.Lock()   // exclusive lock, even for reads
    defer c.mu.Unlock()
    return c.data[key]
}

// sync.RWMutex: read-write lock
// - MULTIPLE goroutines can hold RLock simultaneously
// - Only ONE goroutine can hold Lock (exclusive write lock)
// - Write lock waits for all readers to finish
// - Use when reads outnumber writes significantly

type ReadHeavyCache struct {
    mu   sync.RWMutex
    data map[string]int
}

func (c *ReadHeavyCache) Get(key string) int {
    c.mu.RLock()   // shared read lock — multiple goroutines can hold this
    defer c.mu.RUnlock()
    return c.data[key]
}

func (c *ReadHeavyCache) Set(key string, val int) {
    c.mu.Lock()    // exclusive write lock — all readers must finish first
    defer c.mu.Unlock()
    c.data[key] = val
}
```

**Performance comparison:**

| Scenario | Mutex | RWMutex |
|----------|-------|---------|
| Write-heavy (>30% writes) | Better (less overhead) | Lock contention overhead |
| Read-heavy (<10% writes) | Bottleneck (serializes reads) | Better (concurrent reads) |
| Equal reads/writes | OK | Similar performance |

```go
// Benchmark to illustrate:
// BenchmarkMutexReadOnly    100ns/op   (serialized reads)
// BenchmarkRWMutexReadOnly   20ns/op   (concurrent reads, 10 goroutines)

// Important: RWMutex has writer starvation protection
// A waiting writer prevents new readers from acquiring RLock
// This prevents writers from being blocked indefinitely
```

---

## Answer 8: Implement the fan-out/fan-in pattern

```go
package main

import (
    "context"
    "fmt"
    "sync"
)

// Fan-out: distribute work from one channel to multiple workers
// Fan-in: merge results from multiple channels into one

// Stage 1: Generate items
func generate(ctx context.Context, items ...int) <-chan int {
    out := make(chan int, len(items))
    go func() {
        defer close(out)
        for _, item := range items {
            select {
            case out <- item:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Stage 2: Process — this is the fanned-out stage
// Each call creates an independent worker
func process(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range in {
            select {
            case out <- v * v: // square each number
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Fan-in: merge multiple channels into one
func fanIn(ctx context.Context, channels ...<-chan int) <-chan int {
    merged := make(chan int)
    var wg sync.WaitGroup

    // Forward from each channel to merged
    forward := func(ch <-chan int) {
        defer wg.Done()
        for {
            select {
            case v, ok := <-ch:
                if !ok {
                    return
                }
                select {
                case merged <- v:
                case <-ctx.Done():
                    return
                }
            case <-ctx.Done():
                return
            }
        }
    }

    wg.Add(len(channels))
    for _, ch := range channels {
        go forward(ch)
    }

    // Close merged when all inputs are done
    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Generate numbers 1–12
    nums := generate(ctx, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12)

    // Fan-out: 3 workers process the numbers
    worker1 := process(ctx, nums)
    worker2 := process(ctx, nums)
    worker3 := process(ctx, nums)

    // Fan-in: merge results
    for result := range fanIn(ctx, worker1, worker2, worker3) {
        fmt.Println(result)
    }
}
```

---

## Answer 9: How does the Go race detector work?

The race detector is a tool built into the Go toolchain that instruments memory accesses at compile time and detects races at runtime.

**How it works:**
1. `go build -race` compiles the program with instrumentation code inserted around every memory access
2. At runtime, the instrumentation records which goroutine last accessed each memory location
3. If two goroutines access the same memory concurrently (no synchronization) and at least one is a write — a race is reported

```go
// Example program with a data race:
package main

import "fmt"

func main() {
    count := 0
    done := make(chan bool)

    go func() {
        count++ // Write from goroutine 1
        done <- true
    }()

    count++ // Write from main goroutine — CONCURRENT WRITE!
    <-done
    fmt.Println(count)
}
```

```bash
# Run with race detector:
$ go run -race main.go

==================
WARNING: DATA RACE
Write at 0x00c000016080 by goroutine 6:
  main.main.func1()
      /tmp/main.go:10 +0x40

Previous write at 0x00c000016080 by main goroutine:
  main.main()
      /tmp/main.go:14 +0x84

Goroutine 6 (running) created at:
  main.main()
      /tmp/main.go:9 +0x74
==================
```

**Performance overhead:** 5–10× slowdown, 5–10× more memory. Use for:
- Local development: `go test -race ./...`
- CI pipelines on all PRs
- NOT for production (unless you're debugging a specific race)

---

## Answer 10: What guarantees does the Go memory model provide?

The Go memory model defines when a write to a variable in one goroutine is guaranteed to be observed by a read in another goroutine. The model is based on "happens-before" relationships.

**Happens-before is established by:**

```go
// 1. Goroutine creation: the go statement happens-before the goroutine body
x := 42
go func() {
    fmt.Println(x) // guaranteed to see x=42 (go statement h-b goroutine start)
}()

// 2. Channel send happens-before channel receive (unbuffered)
var msg string
ch := make(chan struct{})
go func() {
    msg = "hello"   // write
    ch <- struct{}{} // send h-b receive — guarantees msg is visible
}()
<-ch
fmt.Println(msg) // guaranteed to see "hello"

// 3. sync.Mutex: Unlock happens-before subsequent Lock
var mu sync.Mutex
var data int

go func() {
    mu.Lock()
    data = 42
    mu.Unlock() // unlock h-b next lock
}()

mu.Lock()
fmt.Println(data) // guaranteed to see 42 (if the goroutine ran first)
mu.Unlock()

// 4. sync.WaitGroup.Done happens-before Wait returns
var wg sync.WaitGroup
var results []int

wg.Add(1)
go func() {
    defer wg.Done()
    results = append(results, 1, 2, 3) // write h-b Done h-b Wait
}()

wg.Wait()
fmt.Println(results) // guaranteed to see [1,2,3]

// 5. sync.Once.Do completion happens-before any subsequent Once.Do call
var once sync.Once
var initialized bool
once.Do(func() { initialized = true })
// Any goroutine calling once.Do after the first sees initialized=true
```

**What is NOT guaranteed (undefined behavior):**
```go
// Unsynchronized concurrent access — DO NOT rely on this
var x int
go func() { x = 1 }()
go func() { x = 2 }()
// Reading x might see 0, 1, 2, or garbage
```
