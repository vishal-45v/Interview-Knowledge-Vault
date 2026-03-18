# Go Performance & Memory — Follow-Up Traps

These are the unexpected behaviors and common mistakes that interviewers probe after you give a correct initial answer.

---

## Trap 1: Pointer to Local Variable — When Does It Actually Matter?

**The trap:** "Returning a pointer to a local variable causes it to escape to the heap. Is that always a problem?"

**The nuance:** It depends on allocation frequency and size. The compiler allocates the variable on the heap — that's not inherently bad. It becomes a problem when:
- The function is called millions of times per second (allocation pressure)
- The object is small and short-lived (pool reuse makes more sense)
- GC pause sensitivity is a requirement

```go
// Fine: rare call, large struct, useful to avoid copy
func NewConfig() *Config {
    c := Config{Timeout: 30 * time.Second, MaxRetries: 3}
    return &c // escapes, but called once at startup
}

// Problematic: called in hot loop
func newPoint(x, y float64) *Point {
    return &Point{x, y} // escapes — 1M allocs/sec in loop!
}

// Fixed: pass by value or reuse
func addPoints(dst []Point, x, y float64) []Point {
    return append(dst, Point{x, y}) // no heap alloc for the Point
}
```

**Key insight:** The question isn't "does it escape?" but "how often is it allocated and collected?"

---

## Trap 2: sync.Pool Objects Are Evicted by GC — Don't Rely on Persistence

**The trap:** "I use sync.Pool as a cache to avoid re-creating expensive objects."

**The problem:** The GC clears the pool at every GC cycle. If your GC runs every few seconds, pool objects survive. But under memory pressure with frequent GC, pool hit rate drops dramatically.

```go
// WRONG: using sync.Pool as a cache
var connPool = sync.Pool{
    New: func() interface{} {
        return createExpensiveDBConnection() // heavy operation!
    },
}

// Get() may call New() every GC cycle under pressure
conn := connPool.Get().(DBConn)
defer connPool.Put(conn)
```

**Correct use cases for sync.Pool:**
- Temporary byte buffers
- Encoder/decoder objects
- Request/response wrapper objects

**For actual connection caches:** use `database/sql` built-in pool, or implement your own with a channel-based pool.

```go
// Proper pool for buffers (eviction is acceptable)
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 4096)
    },
}
```

---

## Trap 3: Interface Conversions Allocate — Boxing Small Values

**The trap:** "Storing a small int in an interface{} is fine, right? It's just one word."

**The problem:** An interface holds two words: (type pointer, value pointer). Small scalar values (int, bool, small structs) must be boxed — a heap allocation occurs to store the value.

```go
// This allocates on every call:
func acceptAny(v interface{}) { /* ... */ }

x := 42
acceptAny(x) // boxes int → heap allocation

// Measure it:
func BenchmarkBoxing(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var v interface{} = i  // allocation each iteration
        _ = v
    }
}
```

**Exceptions — no allocation when:**
- The value is a pointer (fits in one interface word)
- The value is nil
- The value fits in a pointer-sized word AND the compiler applies scalar optimization

**Fix:** Avoid `interface{}` (or `any`) in hot paths. Use generics (Go 1.18+) or concrete types.

---

## Trap 4: Sub-Slice Retains Entire Backing Array (Memory Leak)

**The trap:** "I sliced a large buffer to get just the header bytes. Surely the large buffer is GC'd?"

**The problem:** A sub-slice shares the backing array. The GC considers the entire array live as long as any slice references it.

```go
func processLargeLog(data []byte) []byte {
    // data is 10MB
    header := data[:128] // header is 128 bytes
    return header         // backing array: still 10MB — NOT collected!
}

// Fix: copy to break the reference
func processLargeLog(data []byte) []byte {
    header := data[:128]
    result := make([]byte, len(header))
    copy(result, header)
    return result // now only 128 bytes kept alive
}

// Or use a one-liner:
result := append([]byte(nil), data[:128]...)
```

The same applies to strings (though strings are immutable, sub-strings share storage).

---

## Trap 5: []byte ↔ string Conversion — When It Allocates

**The trap:** "Converting string to []byte always allocates, so I'll use unsafe to avoid it."

**The nuance:**

```go
// Typically allocates (defensive copy)
b := []byte(someString)

// Typically allocates
s := string(someBytes)

// Compiler optimization: NO allocation when converting string→[]byte
// for a map lookup, or in certain range loops
m := map[string]int{"key": 1}
_ = m[string(b)] // may be optimized to avoid alloc

// Force zero-copy (unsafe — only safe if you won't modify the bytes)
import "unsafe"

func stringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

func bytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```

**Warning:** The unsafe approach breaks if the original string is ever collected or the bytes are modified. Only use in read-only, non-concurrent code paths.

---

## Trap 6: Struct Field Ordering — Largest to Smallest Is Not Always Right

**The trap:** "Always order struct fields largest to smallest for minimum padding."

**The nuance:** There's more to it:

```go
// True: sorting by alignment reduces padding
type Good struct {
    F64 float64 // 8 bytes, offset 0
    I32 int32   // 4 bytes, offset 8
    I16 int16   // 2 bytes, offset 12
    I8  int8    // 1 byte,  offset 14
    B   bool    // 1 byte,  offset 15
}                // total: 16 bytes

// But: consider cache line locality for hot fields
// If F64 and B are always accessed together, they should be adjacent.
// If a struct spans multiple cache lines, consider padding to align hot fields:

type CacheOptimized struct {
    // Hot path: fits in one cache line (64 bytes)
    Counter   int64
    LastSeen  int64
    Active    bool
    _         [7]byte // explicit padding

    // Cold path: second cache line
    Metadata  string
    Tags      []string
}
```

Also, `sync.Mutex` should NOT be padded to avoid false sharing — use `sync/atomic` with padding:

```go
type PaddedCounter struct {
    _   [64]byte // cache line padding (false sharing prevention)
    val int64
    _   [56]byte // fill remainder of cache line
}
```

---

## Trap 7: fmt.Sprintf Is Significantly Slower Than strconv

**The trap:** "I use fmt.Sprintf for all my number-to-string conversions. It's readable."

**The reality:**

```go
// Benchmark results (approximate):
// fmt.Sprintf("%d", n)    → ~100 ns/op, 1 alloc/op
// strconv.Itoa(n)         → ~12 ns/op,  1 alloc/op
// strconv.AppendInt(b, n) → ~5 ns/op,   0 allocs/op (to existing buf)

// Prefer strconv in hot paths:
n := 42

// Readable but slow:
s := fmt.Sprintf("%d", n)

// Faster:
s := strconv.Itoa(n)

// Fastest (zero allocation if buf has capacity):
var buf [20]byte
b := strconv.AppendInt(buf[:0], int64(n), 10)
s := string(b)
```

`fmt.Sprintf` uses reflection internally (`interface{}` args), adding overhead.

---

## Trap 8: Goroutine Stack Starts at 2KB — The Initial Allocation Surprise

**The trap:** "Goroutines are cheap, right? I'll spawn 100,000 of them."

**The reality:**

```go
// Each goroutine costs at minimum:
// - 2KB stack (may grow to MB for deep call chains)
// - ~3KB scheduler overhead (G struct in runtime)
// - Any closured variables

// 100,000 goroutines minimum: 100,000 × 5KB ≈ 500MB just for stacks

// Diagnose:
go http.HandleFunc("/goroutines", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "goroutines: %d\n", runtime.NumGoroutine())
})

// Fix: use a worker pool
func workerPool(jobs <-chan Job, numWorkers int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                process(job)
            }
        }()
    }
    wg.Wait()
}
```

Also note: stack growth (copying) costs CPU. Functions with many stack allocations in recursive patterns can trigger frequent stack copies.

---

## Trap 9: runtime.GC() Does NOT Clear sync.Pool

**The trap:** "I call runtime.GC() in my test to reset state, including clearing sync.Pool."

**The reality:** In Go 1.13+, sync.Pool implements a two-generation scheme. `runtime.GC()` marks current pool entries as "old." A second GC cycle is needed to actually collect them.

```go
// In tests, to reliably clear sync.Pool:
func clearPool() {
    runtime.GC()
    runtime.GC() // two GC cycles needed to fully evict pool
}

// Even better: use runtime/debug.FreeOSMemory
import "runtime/debug"
debug.FreeOSMemory() // triggers GC and returns memory to OS
```

This matters when writing tests that verify allocation counts — the pool may not be clean after a single GC.

---

## Trap 10: CGO Disables Key GC Optimizations

**The trap:** "I'll call a C library via cgo for performance."

**The problem:** CGO has significant overhead:

```go
// Each cgo call costs:
// - ~60-200 ns (vs ~5 ns for a Go function call)
// - Thread switch from goroutine scheduler to OS thread
// - GC must pause and scan C stack frames
// - C pointers cannot be moved by Go GC

// CGO also disables:
// - Stack copying for goroutines calling C (must use OS thread)
// - Some escape analysis optimizations
// - Cross-compilation (CGO_ENABLED=0 required for cross-compile)

// If using CGO-heavy code, avoid calling it in tight loops
// Batch cgo calls and do more work per call:
for _, item := range items {
    cResult := C.processOne(item) // bad: N cgo calls
}

// Better: pass a batch to C
C.processBatch((*C.Item)(unsafe.Pointer(&items[0])), C.int(len(items)))
```

---

## Trap 11: timer.After Leaks Memory Inside Loops

**The trap:** "I use time.After(timeout) inside a for loop for per-iteration timeouts."

**The problem:**

```go
// MEMORY LEAK: each time.After creates a timer that is NOT collected
// until it fires, even if the select case is never chosen
for {
    select {
    case msg := <-ch:
        process(msg)
    case <-time.After(5 * time.Second): // new timer created every iteration!
        log.Println("slow message")
    }
}
```

**Fix:** Reuse a `time.Timer`:

```go
timer := time.NewTimer(5 * time.Second)
defer timer.Stop()

for {
    timer.Reset(5 * time.Second) // reuse
    select {
    case msg := <-ch:
        if !timer.Stop() {
            <-timer.C // drain channel if timer fired
        }
        process(msg)
    case <-timer.C:
        log.Println("slow message")
    }
}
```

---

## Trap 12: reflect.DeepEqual Is Much Slower Than You Think

**The trap:** "I use reflect.DeepEqual in my hot path to compare structs."

**The reality:**

```go
// reflect.DeepEqual:
// - Uses reflection (100x slower than direct comparison)
// - Allocates interface values
// - Traverses nested structures recursively

// Benchmark:
// reflect.DeepEqual → ~500 ns/op, allocations
// manual == comparison → ~2 ns/op, 0 allocs

// Fix: implement Equals method
type Point struct{ X, Y float64 }

func (p Point) Equal(other Point) bool {
    return p.X == other.X && p.Y == other.Y
}

// Or use == directly (works for comparable types)
p1 := Point{1, 2}
p2 := Point{1, 2}
fmt.Println(p1 == p2) // true, no reflection

// For slices/maps: implement custom comparison
func sliceEqual(a, b []int) bool {
    if len(a) != len(b) {
        return false
    }
    for i := range a {
        if a[i] != b[i] {
            return false
        }
    }
    return true
}
```

Reserve `reflect.DeepEqual` for tests only, never production hot paths.
