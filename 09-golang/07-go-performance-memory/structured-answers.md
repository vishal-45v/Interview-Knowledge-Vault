# Go Performance & Memory — Structured Answers

---

## Answer 1: How Does Go's Garbage Collector Work?

**One-sentence answer:** Go uses a concurrent tri-color mark-and-sweep GC that runs mostly alongside your program, with two very short stop-the-world pauses per cycle.

**Full explanation:**

The GC runs in three phases:

**Phase 1 — Mark Setup (STW, ~100 µs):**
- Stop all goroutines briefly
- Enable write barriers on pointer writes
- Scan goroutine stacks and globals to find root pointers
- Mark roots grey

**Phase 2 — Concurrent Mark:**
- Resume goroutines (application and GC run concurrently)
- GC goroutines process the grey set: scan each grey object's references, mark them grey, mark the parent black
- Write barriers ensure new pointer writes are tracked
- "Assist" mechanism: goroutines allocating memory must assist the GC mark work

**Phase 3 — Mark Termination (STW, ~100 µs):**
- Stop all goroutines
- Verify no grey objects remain
- Disable write barriers
- Resume goroutines

**Phase 4 — Concurrent Sweep:**
- Free white (unreachable) objects
- Runs concurrently, returning memory to the heap allocator

**GC trigger:**
```go
// Triggered when: heap size >= live_heap_after_last_GC × (1 + GOGC/100)
// Default GOGC=100: triggers when heap doubles
// Tune with:
import "runtime/debug"
debug.SetGCPercent(200)         // less frequent
debug.SetMemoryLimit(512 << 20) // absolute ceiling (Go 1.19+)
```

**Monitoring GC:**
```bash
GODEBUG=gctrace=1 ./myapp
# Output per cycle:
# gc 1 @0.023s 2%: 0.014+0.15+0.024 ms clock, 0.028+0.12/0.14/0.022+0.048 ms cpu, 4->4->2 MB, 5 MB goal, 8 P
```

---

## Answer 2: What Is Escape Analysis and How Does It Affect Performance?

**One-sentence answer:** Escape analysis is a compile-time analysis that decides whether a variable can live on the goroutine's stack (fast, free) or must be allocated on the GC-managed heap (slower, creates GC pressure).

**Stack vs heap:**

| Stack | Heap |
|-------|------|
| Allocated on function entry | Allocated by runtime.mallocgc |
| Freed automatically on return | Freed by GC |
| ~0 ns allocation cost | ~20-50 ns allocation cost |
| Cache-friendly (sequential) | Scattered across memory |

**What causes escape:**

```go
// 1. Returning a pointer
func New() *T {
    t := T{}
    return &t  // t escapes: lives beyond its function
}

// 2. Storing in interface
func log(v interface{}) {} // argument may escape

// 3. Goroutine closure
func spawn(x int) {
    go func() { fmt.Println(x) }() // x escapes to heap
}

// 4. Slice/map elements if the container escapes
func buildSlice() []*Item {
    items := make([]*Item, 10)
    for i := range items {
        items[i] = &Item{i}  // Item escapes (stored in escaping slice)
    }
    return items
}
```

**Checking escape analysis:**
```bash
go build -gcflags="-m=1" ./...
# ./main.go:10:6: &t escapes to heap

go build -gcflags="-m=2" ./...  # more verbose — shows why
```

**Performance impact:**

```go
// Benchmark: stack vs heap alloc
func BenchmarkStack(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        p := Point{1, 2}   // stack — 0 allocs
        _ = p.X + p.Y
    }
}

func BenchmarkHeap(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        p := &Point{1, 2}  // heap — 1 alloc/op
        _ = p.X + p.Y
    }
}
// Stack: ~0.5 ns/op, 0 allocs
// Heap:  ~15 ns/op,  1 alloc/op
```

---

## Answer 3: How Do You Profile a Go Application with pprof?

**One-sentence answer:** Import `net/http/pprof`, expose the debug endpoint, then collect and analyze profiles with `go tool pprof`.

**Setup:**

```go
package main

import (
    "net/http"
    _ "net/http/pprof" // registers handlers on DefaultServeMux
)

func main() {
    // Start pprof server (separate port from app)
    go http.ListenAndServe(":6060", nil)

    // Start your application
    runApp()
}
```

**Collecting profiles:**

```bash
# CPU profile (30 seconds)
curl "http://localhost:6060/debug/pprof/profile?seconds=30" -o cpu.prof

# Heap (in-use memory)
curl "http://localhost:6060/debug/pprof/heap" -o heap.prof

# All allocations (since start)
curl "http://localhost:6060/debug/pprof/allocs" -o allocs.prof

# Goroutines
curl "http://localhost:6060/debug/pprof/goroutine" -o goroutine.prof

# Blocking events
curl "http://localhost:6060/debug/pprof/block" -o block.prof

# Mutex contention
curl "http://localhost:6060/debug/pprof/mutex" -o mutex.prof
```

**Enable block and mutex profiling (not on by default):**
```go
runtime.SetBlockProfileRate(1)     // record all blocking events
runtime.SetMutexProfileFraction(1) // record all mutex contention
```

**Analyzing:**
```bash
# Interactive terminal
go tool pprof cpu.prof
(pprof) top10
(pprof) top10 -cum        # include caller overhead
(pprof) list mypackage.MyFunc
(pprof) disasm MyFunc     # assembly-level view

# Web UI (best for flame graphs)
go tool pprof -http=:8080 cpu.prof

# From test:
go test -bench=BenchmarkFoo -cpuprofile=cpu.prof -memprofile=mem.prof
```

**In-process profiling (e.g., for CLIs):**
```go
func main() {
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer func() {
        pprof.StopCPUProfile()
        f.Close()
    }()
    // ... work
}
```

---

## Answer 4: What Is sync.Pool and How Do You Use It to Reduce GC Pressure?

**One-sentence answer:** `sync.Pool` is a goroutine-safe cache of reusable objects that reduces allocations by recycling objects instead of letting the GC collect and re-allocate them.

**How it works:**

```go
var pool = sync.Pool{
    New: func() interface{} {
        // Called when pool is empty
        return &MyObject{buf: make([]byte, 0, 1024)}
    },
}

func handleRequest(data []byte) {
    // Get from pool (or create if empty)
    obj := pool.Get().(*MyObject)

    // ALWAYS reset before use — pool may return dirty object
    obj.Reset()

    // Use the object
    obj.Process(data)

    // Return to pool when done
    pool.Put(obj)
}
```

**Real-world example with HTTP handlers:**

```go
var jsonEncoderPool = sync.Pool{
    New: func() interface{} {
        return json.NewEncoder(io.Discard) // placeholder writer
    },
}

func writeJSON(w http.ResponseWriter, v interface{}) error {
    w.Header().Set("Content-Type", "application/json")

    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)

    enc := json.NewEncoder(buf)
    if err := enc.Encode(v); err != nil {
        return err
    }

    _, err := w.Write(buf.Bytes())
    return err
}
```

**Rules:**

1. The pool may return an object from any goroutine — always reset before use
2. Objects can be evicted (GC'd) at any time — do not store persistent state
3. Do not pool connections or file handles — use dedicated pools for those
4. Return the object to the pool via `defer` to ensure it happens even on panic

**Verify it helps:**

```go
func BenchmarkWithPool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        obj := pool.Get().(*MyObject)
        obj.Reset()
        obj.DoWork()
        pool.Put(obj)
    }
}
// Compare: with pool → 0 allocs/op vs without → 1 alloc/op
```

---

## Answer 5: How Do You Pre-allocate Slices and Maps to Avoid Reallocations?

**One-sentence answer:** Pass a capacity hint to `make()` when you know or can estimate the final size, so the runtime allocates memory once instead of repeatedly doubling the slice.

**Slice growth behavior without pre-allocation:**

```go
// Each append that exceeds capacity triggers:
// 1. Allocate new array (2× capacity)
// 2. Copy all elements
// 3. GC the old array

// For 1000 elements: ~10 reallocations, ~2000 total elements copied
```

**Pre-allocating slices:**

```go
// Known size: use length
n := 10000
result := make([]int, n) // length=n, capacity=n
for i := 0; i < n; i++ {
    result[i] = i * i
}

// Known capacity but built incrementally: use cap only
result := make([]int, 0, n)
for i := 0; i < n; i++ {
    result = append(result, i*i) // never reallocates
}

// Unknown but estimated: over-provision then trim
result := make([]Record, 0, estimatedSize*2)
for _, item := range items {
    if item.Valid() {
        result = append(result, item.ToRecord())
    }
}
result = result[:len(result):len(result)] // trim excess capacity
```

**Pre-allocating maps:**

```go
// Bad: map grows from 0, multiple rehashings
m := make(map[string]int)
for _, k := range keys {
    m[k]++
}

// Good: hint avoids most rehashing
m := make(map[string]int, len(keys))
for _, k := range keys {
    m[k]++
}
```

Note: the map hint is advisory, not exact. The map may still resize if many entries hash to the same bucket.

**Benchmark to demonstrate:**

```go
func BenchmarkSliceNoPrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := []int{}
        for j := 0; j < 10000; j++ {
            s = append(s, j)
        }
    }
}

func BenchmarkSlicePrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := make([]int, 0, 10000)
        for j := 0; j < 10000; j++ {
            s = append(s, j)
        }
    }
}
// Pre-alloc: ~3× faster, 1 alloc vs ~14 allocs
```

---

## Answer 6: What Is Struct Padding and How Do You Optimize Struct Layout?

**One-sentence answer:** Go inserts padding bytes between struct fields to ensure each field is aligned to its natural alignment boundary; re-ordering fields largest-to-smallest minimizes wasted bytes.

**Why alignment matters:**

CPU architectures require data to be loaded from aligned addresses:
- `int64` must be at addresses divisible by 8
- `int32` must be at addresses divisible by 4
- `bool`/`byte` can be at any address

**Example of bad layout:**

```go
type Wasteful struct {
    Flag    bool    // 1 byte at offset 0
    // 7 bytes padding inserted here
    Value   float64 // 8 bytes at offset 8
    Count   int32   // 4 bytes at offset 16
    // 4 bytes padding inserted here
    Mode    byte    // 1 byte at offset 24
    // 7 bytes padding to align struct to 8
}
// Total: 32 bytes

import "unsafe"
fmt.Println(unsafe.Sizeof(Wasteful{})) // 32
```

**Optimized layout (sort descending by size):**

```go
type Efficient struct {
    Value   float64 // 8 bytes at offset 0
    Count   int32   // 4 bytes at offset 8
    Mode    byte    // 1 byte at offset 12
    Flag    bool    // 1 byte at offset 13
    // 2 bytes padding to align struct to 8
}
// Total: 16 bytes — saves 16 bytes (50%)!

fmt.Println(unsafe.Sizeof(Efficient{})) // 16
```

**Tooling:**

```bash
# Install the analyzer
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest

# Find suboptimal structs (dry run)
fieldalignment ./...

# Auto-fix
fieldalignment -fix ./...
```

**Practical impact:**

If you have 10 million `Wasteful` structs in memory:
- Before: 10M × 32 bytes = 320 MB
- After: 10M × 16 bytes = 160 MB

Beyond size: keeping related hot fields in the same cache line (64 bytes) reduces cache misses.

---

## Answer 7: How Do You Reduce Allocations When Building Strings?

**One-sentence answer:** Use `strings.Builder` with `Grow()` for assembly, `strconv` instead of `fmt.Sprintf` for numbers, and `[]byte` buffers for heavy formatting.

**String concatenation cost:**

```go
// strings are immutable: each + creates a new allocation
s := ""
for i := 0; i < 1000; i++ {
    s += "x" // 1000 allocations, copying 0+1+2+...+999 = O(n²) work
}
```

**strings.Builder (best for dynamic assembly):**

```go
var sb strings.Builder
sb.Grow(1000) // pre-allocate if size is known

for i := 0; i < 1000; i++ {
    sb.WriteByte('x')
}
result := sb.String() // single allocation for the final string
```

**strconv vs fmt for numbers:**

```go
// fmt.Sprintf: ~100ns, uses reflection
s := fmt.Sprintf("%d", 42)

// strconv.Itoa: ~12ns, no reflection
s := strconv.Itoa(42)

// strconv.AppendInt: ~5ns, zero allocation (appends to existing buffer)
buf := make([]byte, 0, 20)
buf = strconv.AppendInt(buf, 42, 10)
```

**bytes.Buffer vs strings.Builder:**

```go
// bytes.Buffer: implements io.Reader and io.Writer — use for I/O
var buf bytes.Buffer
buf.WriteString("hello ")
buf.WriteString("world")
_, err := w.Write(buf.Bytes())

// strings.Builder: write-only, String() is zero-copy — use for string assembly
var sb strings.Builder
sb.WriteString("hello ")
sb.WriteString("world")
return sb.String()
```

**fmt.Fprintf to a Builder (readable + fast):**

```go
var sb strings.Builder
sb.Grow(256)
fmt.Fprintf(&sb, "user=%s age=%d score=%.2f\n", user.Name, user.Age, user.Score)
// fmt.Fprintf is slow, but if readability matters, at least you avoid the final alloc
```

---

## Answer 8: What Do GOGC and GOMEMLIMIT Control?

**GOGC:**

`GOGC` sets the GC trigger threshold as a percentage of live heap growth.

```bash
# Default: GC when heap doubles
GOGC=100

# Aggressive: GC when heap grows 50% (more CPU, less memory)
GOGC=50

# Lazy: GC when heap grows 300% (less CPU, more memory)
GOGC=300

# Disable GC entirely (dangerous — for benchmarks only)
GOGC=off
```

```go
import "runtime/debug"

// Programmatic control:
debug.SetGCPercent(200) // equivalent to GOGC=200

// Useful patterns:
// - Startup: GOGC=off during initialization, then restore
old := debug.SetGCPercent(-1) // disable
loadHugeCacheFromDisk()
debug.SetGCPercent(old)       // restore
```

**GOMEMLIMIT (Go 1.19+):**

Sets a soft ceiling on total Go runtime memory (heap + runtime overhead).

```bash
GOMEMLIMIT=512MiB ./service
GOMEMLIMIT=1GiB ./service
```

```go
import "runtime/debug"

debug.SetMemoryLimit(512 * 1024 * 1024) // 512 MiB
```

**How they interact:**

```
Normal:   GOGC=100, GOMEMLIMIT=unset
          → GC when heap doubles
          → Memory grows unbounded (OOM risk)

Latency:  GOGC=100, GOMEMLIMIT=512MiB
          → GC when heap doubles OR when approaching 512MiB
          → Prevents OOM, normal GC frequency otherwise

Throughput: GOGC=off, GOMEMLIMIT=512MiB
          → No GC until approaching 512MiB
          → Then GC aggressively to stay under limit
          → Maximum throughput, controlled memory ceiling
```

**Recommended production setting:**

```bash
# Set limit to ~90% of available memory
GOMEMLIMIT=900MiB GOGC=100 ./service
```

---

## Answer 9: How Do You Detect and Fix Memory Leaks in Go?

**One-sentence answer:** Use pprof heap profiles taken at intervals to find what's accumulating, then trace to the root cause (forgotten goroutine, unbounded cache, retained sub-slice, etc.).

**Detection workflow:**

```bash
# Step 1: Take baseline
curl http://localhost:6060/debug/pprof/heap -o heap_t0.prof

# Step 2: Wait for memory to grow
sleep 3600  # or use time-based automation

# Step 3: Take another profile
curl http://localhost:6060/debug/pprof/heap -o heap_t1.prof

# Step 4: Diff the profiles
go tool pprof -base=heap_t0.prof heap_t1.prof
(pprof) top10 -inuse_space  # what grew?
```

**Common leak patterns and fixes:**

```go
// Leak 1: Goroutine blocked on channel with no sender
func startWorker(data <-chan []byte) {
    go func() {
        for d := range data { // goroutine leaks if data is never closed
            process(d)
        }
    }()
}
// Fix: always close the channel when done, or use context

// Leak 2: Unbounded in-memory map/cache
var cache = map[string]*Result{}
var mu sync.RWMutex

func getOrCompute(key string) *Result {
    mu.Lock()
    defer mu.Unlock()
    if r, ok := cache[key]; ok {
        return r
    }
    r := compute(key)
    cache[key] = r // grows forever!
    return r
}
// Fix: use an LRU cache with bounded size
// github.com/hashicorp/golang-lru or sync.Map with eviction

// Leak 3: time.After in loop (timer not collected until it fires)
for {
    select {
    case <-ch:
    case <-time.After(30 * time.Second): // new timer every iteration!
    }
}
// Fix: use time.NewTimer + Reset

// Leak 4: http.Response.Body not closed
resp, _ := http.Get(url)
// Forgot: defer resp.Body.Close() — goroutine and memory leak!
```

**Using goleak in tests:**

```go
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

// Or per-test:
func TestFoo(t *testing.T) {
    defer goleak.VerifyNone(t)
    // test body
}
```

---

## Answer 10: What Is the Overhead of Using Interfaces vs Concrete Types?

**One-sentence answer:** Interface method calls are ~2–5× slower than concrete type calls due to a vtable (itab) lookup and pointer indirection, plus potential heap allocations from boxing non-pointer values.

**Interface call mechanics:**

An interface value holds two words:
1. `*itab` — pointer to type + method table
2. `data` — pointer to (or copy of) the value

A method call via interface: look up the function pointer in the itab → dereference data pointer → call. This prevents inlining (compiler doesn't know the concrete type at compile time).

**Benchmark:**

```go
type Adder interface {
    Add(a, b int) int
}

type ConcreteAdder struct{}
func (c ConcreteAdder) Add(a, b int) int { return a + b }

var globalResult int

func BenchmarkConcrete(b *testing.B) {
    c := ConcreteAdder{}
    for i := 0; i < b.N; i++ {
        globalResult = c.Add(i, i+1)
    }
}

func BenchmarkInterface(b *testing.B) {
    var a Adder = ConcreteAdder{}
    for i := 0; i < b.N; i++ {
        globalResult = a.Add(i, i+1)
    }
}
// Concrete: ~0.5 ns/op (inlined by compiler, just: globalResult = i + i + 1)
// Interface: ~2-4 ns/op (itab lookup, indirect call, no inlining)
```

**When to accept the overhead:**
- Testability (dependency injection via interface) is worth it in non-hot paths
- The function body is long enough that the call overhead is negligible
- The abstraction provides meaningful extensibility

**When to avoid interfaces:**
- Inner loops processing millions of items
- Allocator/buffer paths (use concrete types + generics)

```go
// Faster alternative with generics (Go 1.18+):
func ProcessAll[T Processor](items []T) {
    for _, item := range items {
        item.Process()
        // Compiler can devirtualize and inline if T is known at call site
    }
}
```
