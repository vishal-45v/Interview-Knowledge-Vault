# Go Performance & Memory — Theory Questions

---

## Garbage Collector

**Q1. Describe Go's garbage collector algorithm. What is tri-color mark-and-sweep?**

Go uses a concurrent, tri-color mark-and-sweep garbage collector. Every heap object is assigned one of three colors:

- **White** — not yet visited; candidate for collection
- **Grey** — discovered (reachable) but children not yet scanned
- **Black** — fully scanned; all references followed

The GC starts by marking all roots (globals, stack variables) grey. It then processes grey objects: scans their references, marks those white children grey, then marks the parent black. When no grey objects remain, all white objects are unreachable and are swept (memory reclaimed). The key invariant: a black object must never point directly to a white object (the *tri-color invariant*).

---

**Q2. What are write barriers in Go's GC and why are they necessary?**

Because Go's GC runs concurrently with application goroutines (mutators), a running goroutine can modify the object graph while the GC is scanning. Without protection, a goroutine could:
1. Store a reference to a white object inside a black object (breaking the invariant)
2. Delete the only grey reference to that white object

The **write barrier** is a small hook inserted by the compiler around pointer writes. When a pointer store occurs during GC, the barrier records the old and/or new pointer value so the GC can re-shade objects to grey. Go uses a *hybrid write barrier* (since Go 1.17) combining Dijkstra (new pointer) and Yuasa (old pointer) barriers, ensuring correctness without a stop-the-world re-scan phase.

---

**Q3. What are GC pauses in Go and how long are they typically?**

Go's GC has two small stop-the-world (STW) pauses per cycle:
1. **GC start** — enables write barriers, stacks are scanned
2. **GC end (mark termination)** — final scan to ensure no grey objects remain

Since Go 1.14+, these pauses are sub-millisecond (typically 100–500 µs) even for large heaps. The bulk of the GC work (marking) happens concurrently while your program runs. You can observe pause times with `GODEBUG=gccheckmark=1` or by looking at runtime metrics.

---

**Q4. What does the GOGC environment variable control?**

`GOGC` sets the **GC target percentage**. It means: trigger GC when live heap size grows by `GOGC%` since the last GC cycle.

- Default: `GOGC=100` — GC triggers when heap doubles
- `GOGC=50` — more frequent GC, less memory overhead, more CPU
- `GOGC=200` — less frequent GC, more memory, less CPU
- `GOGC=off` — disables GC entirely (dangerous, for benchmarks only)

```go
// Programmatically set GC percentage
import "runtime/debug"

debug.SetGCPercent(200) // less frequent GC for throughput-sensitive code
```

---

**Q5. What is GOMEMLIMIT and how does it differ from GOGC?**

`GOMEMLIMIT` (Go 1.19+) sets a **soft memory limit** for the Go runtime's total heap + runtime overhead. Unlike `GOGC` which is relative, `GOMEMLIMIT` is absolute.

```bash
GOMEMLIMIT=512MiB ./myservice
```

When memory approaches the limit, the GC increases its effort to stay under it, even if `GOGC` would not normally trigger a cycle. This prevents OOM kills. It is "soft" because the runtime can temporarily exceed it if truly necessary.

**Key difference:**
- `GOGC=100` controls *frequency* relative to live heap
- `GOMEMLIMIT=512MiB` controls *absolute ceiling* of memory usage

Using both together is the recommended production approach: `GOGC=off GOMEMLIMIT=512MiB` can give the best latency by letting memory grow freely until the ceiling, then GC aggressively.

---

**Q6. What is escape analysis in Go?**

Escape analysis is a compile-time analysis that determines whether a variable can be allocated on the **stack** (cheap, automatically freed) or must be allocated on the **heap** (managed by GC).

A variable "escapes to the heap" when it is reachable beyond its declaring scope:
- Returned as a pointer
- Stored in an interface
- Captured by a goroutine closure
- Too large for the stack

```go
func stackAlloc() int {
    x := 42  // x lives on the stack
    return x  // value copy, x does not escape
}

func heapAlloc() *int {
    x := 42   // x escapes to heap because we return its address
    return &x
}
```

View escape decisions:
```bash
go build -gcflags="-m" ./...
# or more verbose:
go build -gcflags="-m=2" ./...
```

---

**Q7. What factors cause a variable to escape to the heap?**

1. **Returned pointer** — `return &x`
2. **Interface boxing** — `var i interface{} = x` (for non-pointer types)
3. **Stored in a data structure that escapes** — `s.Field = &x` where `s` escapes
4. **Goroutine closure** — `go func() { use(x) }()`
5. **Too large** — variables larger than ~32KB may be heap-allocated
6. **Dynamic size** — `make([]byte, n)` where `n` is not known at compile time

---

**Q8. How do you profile memory in a Go program?**

**Using runtime/pprof (in-process):**

```go
import (
    "os"
    "runtime/pprof"
)

f, _ := os.Create("mem.prof")
defer f.Close()
runtime.GC() // flush finalizers
pprof.WriteHeapProfile(f)
```

**Using net/http/pprof (HTTP endpoint):**

```go
import _ "net/http/pprof"

go http.ListenAndServe(":6060", nil)
```

Then:
```bash
go tool pprof http://localhost:6060/debug/pprof/heap
```

**In tests/benchmarks:**
```bash
go test -bench=. -memprofile=mem.prof
go tool pprof mem.prof
```

---

**Q9. What pprof commands do you use most often when analyzing heap profiles?**

```bash
# Interactive mode
go tool pprof mem.prof

# Inside pprof shell:
(pprof) top10            # top 10 allocating functions
(pprof) top10 -cum       # cumulative (includes callees)
(pprof) list FunctionName  # annotated source
(pprof) web              # open flame graph in browser (requires graphviz)

# Direct web UI (Go 1.10+):
go tool pprof -http=:8080 mem.prof
```

Key profile types:
- `heap` — current live allocations (default: inuse_space)
- `allocs` — total allocations since program start
- `goroutine` — goroutine stack traces
- `block` — goroutine blocking events
- `mutex` — mutex contention

---

**Q10. How do you do CPU profiling in Go?**

```go
import (
    "os"
    "runtime/pprof"
)

f, _ := os.Create("cpu.prof")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
// ... work happens here ...
```

Via HTTP:
```bash
# Collect 30 seconds of CPU profile
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof -http=:8080 cpu.prof
```

Via test:
```bash
go test -bench=BenchmarkFoo -cpuprofile=cpu.prof
go tool pprof cpu.prof
```

---

**Q11. What is sync.Pool and why does it reduce GC pressure?**

`sync.Pool` is a thread-safe cache of temporary objects. Instead of allocating a new object and later GC'ing it, you `Get()` from the pool (reuses existing) and `Put()` back when done.

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process(data []byte) string {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)

    buf.Write(data)
    return buf.String()
}
```

The GC can **evict pool entries at any time** (during STW). This means:
- Pools survive within a GC cycle
- Pools are cleared on each GC
- Do NOT use sync.Pool for things that must persist

---

**Q12. How do you pre-allocate slices and maps to avoid repeated allocations?**

```go
// Bad: grows dynamically, causing multiple reallocations
var result []int
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// Good: single allocation
result := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// Map pre-allocation
m := make(map[string]int, 1000) // hint: ~1000 entries
```

For slices, if the final size is known exactly:
```go
result := make([]int, 10000) // length AND capacity set
for i := range result {
    result[i] = i
}
```

---

**Q13. When does value semantics outperform pointer semantics?**

Value semantics (pass by value) can be faster when:
- The struct is small (fits in a register or cache line — ≤ 64 bytes)
- It avoids heap allocation triggered by pointer escape
- It avoids indirection (pointer dereference = potential cache miss)

```go
type SmallPoint struct { X, Y float64 } // 16 bytes — pass by value

func distanceValue(p SmallPoint) float64 {  // no escape, stack alloc
    return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

func distancePointer(p *SmallPoint) float64 { // p might escape
    return math.Sqrt(p.X*p.X + p.Y*p.Y)
}
```

Rule of thumb: use pointer receivers/args when the struct is large (> 64 bytes) or when mutation is needed. Use value for small, read-only structs.

---

**Q14. How does strings.Builder differ from naive string concatenation?**

String concatenation with `+` creates a new string allocation each time:

```go
// Bad: O(n^2) allocations
s := ""
for i := 0; i < 1000; i++ {
    s += fmt.Sprintf("item%d", i) // new allocation each iteration
}

// Good: O(1) amortized allocations
var sb strings.Builder
sb.Grow(10000) // optional: pre-allocate
for i := 0; i < 1000; i++ {
    fmt.Fprintf(&sb, "item%d", i)
}
result := sb.String()
```

`strings.Builder` maintains a `[]byte` internally and only allocates the final string once. `bytes.Buffer` is similar but also satisfies `io.Reader`.

---

**Q15. What compiler optimizations does Go perform? What is function inlining?**

**Inlining:** The compiler replaces a function call with the function body at the call site, eliminating call overhead and enabling further optimizations.

```go
// Small functions are typically inlined:
func add(a, b int) int { return a + b }

x := add(3, 4) // compiled as: x := 3 + 4
```

View inlining decisions:
```bash
go build -gcflags="-m" .
# Output: ./main.go:5:6: can inline add
```

Functions are NOT inlined if they:
- Are too large (budget exceeded)
- Contain `go`, `defer`, `select`, `for` (with some exceptions)
- Contain `recover()`
- Are recursive

Other optimizations: constant folding, dead code elimination, nil check elimination, bounds check elimination (BCE).

---

**Q16. How do you write and run benchmarks in Go?**

```go
// file: math_test.go
func BenchmarkFibonacci(b *testing.B) {
    b.ReportAllocs() // report allocation count and bytes
    for i := 0; i < b.N; i++ {
        fibonacci(35)
    }
}

// Sub-benchmarks with different sizes:
func BenchmarkSort(b *testing.B) {
    sizes := []int{100, 1000, 10000}
    for _, n := range sizes {
        b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
            data := generateData(n)
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                sort.Ints(data)
            }
        })
    }
}
```

```bash
go test -bench=BenchmarkFibonacci -benchmem -count=5 -benchtime=2s
```

Use `benchstat` to compare before/after:
```bash
go test -bench=. -count=5 > before.txt
# make change
go test -bench=. -count=5 > after.txt
benchstat before.txt after.txt
```

---

**Q17. What is GOMAXPROCS and how does it affect performance?**

`GOMAXPROCS` sets the maximum number of OS threads that can execute Go code simultaneously. Default since Go 1.5: number of logical CPUs.

```go
import "runtime"

runtime.GOMAXPROCS(4) // use 4 OS threads
fmt.Println(runtime.GOMAXPROCS(0)) // query current value
```

```bash
GOMAXPROCS=1 go run main.go  # single-threaded
```

In containerized environments, be aware: Go reads `/proc/cpuinfo` for CPU count, which may show the host's CPU count, not the container's limit. Use `uber-go/automaxprocs` to set `GOMAXPROCS` based on cgroup CPU quota.

```go
import _ "go.uber.org/automaxprocs"
```

---

**Q18. What is struct field alignment and padding in Go?**

CPU architectures require data to be aligned to specific boundaries (e.g., an int64 must be at an 8-byte boundary). The Go compiler inserts **padding bytes** to satisfy alignment, which can waste memory.

```go
// Bad layout: 24 bytes
type Bad struct {
    a bool    // 1 byte + 7 bytes padding
    b float64 // 8 bytes
    c bool    // 1 byte + 7 bytes padding
}

// Good layout: 16 bytes (sort largest to smallest, then bool)
type Good struct {
    b float64 // 8 bytes
    a bool    // 1 byte
    c bool    // 1 byte + 6 bytes padding (struct must align to 8)
}
```

Check sizes:
```go
fmt.Println(unsafe.Sizeof(Bad{}))  // 24
fmt.Println(unsafe.Sizeof(Good{})) // 16
```

Tool: `fieldalignment` from `golang.org/x/tools/go/analysis/passes/fieldalignment`

---

**Q19. What is the performance overhead of the reflect package?**

`reflect` is significantly slower than direct code because:
1. It bypasses compiler optimizations (inlining, escape analysis)
2. It uses interface values (boxing/unboxing)
3. It performs runtime type lookups
4. It accesses memory indirectly

```go
// Direct: ~0.5 ns/op
x := MyStruct{Name: "test"}
_ = x.Name

// Via reflect: ~50-200 ns/op
v := reflect.ValueOf(x)
_ = v.FieldByName("Name").String()
```

Avoid `reflect` in hot paths. Common alternatives:
- Code generation (`go generate` + templates)
- Type assertions with known interfaces
- `encoding/json` with code-gen tools (easyjson, ffjson)

---

**Q20. What is the overhead of defer in Go?**

Since Go 1.14, `defer` in most cases is **open-coded** — the compiler inlines the deferred call at all return sites, making it nearly zero-overhead compared to explicit cleanup.

```go
// Pre-1.14: heap allocation for defer record (~50ns)
// Post-1.14: open-coded, ~2-3ns overhead

func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // essentially free since Go 1.14
    // ...
}
```

`defer` is NOT open-coded (falls back to heap allocation) when:
- Inside a loop
- The function has too many defers (>8)
- Unusual control flow

---

**Q21. How do you diagnose goroutine leaks?**

```go
// Goroutine profile via pprof
http.HandleFunc("/debug/pprof/", pprof.Index) // already included with _ import

// In tests, use goleak:
import "go.uber.org/goleak"

func TestMyFunc(t *testing.T) {
    defer goleak.VerifyNone(t)
    // test code
}
```

Common causes:
- Channel send/receive with no reader/writer and no select default
- `http.Get()` without a timeout (hangs forever)
- Goroutine waiting for a mutex that's never released
- Timer leak: `time.After` inside a loop creates timers that aren't GC'd until fired

---

**Q22. What is the difference between inuse_space and alloc_space in heap profiles?**

- **inuse_space** (default): shows memory **currently allocated** and not yet freed — useful for finding memory leaks and current memory consumers
- **alloc_space**: shows **total bytes allocated** since program start — useful for finding allocation hot spots causing GC pressure

```bash
go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap
```

A function that allocates a lot (`alloc_space`) but releases it quickly may not appear in `inuse_space`.

---

**Q23. How does interface boxing cause allocations?**

When a concrete value is stored in an interface, the Go runtime may allocate on the heap to hold the value if it doesn't fit in a pointer:

```go
var i interface{}

// No allocation: *int fits in interface word (it's a pointer)
x := 42
i = &x

// Allocation: int value must be boxed into a heap-allocated word
i = 42  // heap allocation occurs for the int
```

Escape analysis decides this at compile time. Storing pointer types in interfaces generally avoids extra allocation. This matters in hot paths with many interface conversions.

---

**Q24. What is the runtime/trace package and when do you use it?**

`runtime/trace` captures a detailed execution trace including:
- Goroutine scheduling (created, blocked, unblocked)
- GC events
- Heap size over time
- OS thread events
- Network I/O

```go
import "runtime/trace"

f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()
```

Analyze:
```bash
go tool trace trace.out
```

Use when `pprof` doesn't show the bottleneck — latency issues (scheduling, GC pauses) are invisible in CPU profiles.

---

**Q25. How do you reduce memory fragmentation in Go?**

1. **Use sync.Pool** — reuse objects to avoid fragmentation patterns
2. **Pre-allocate slices** — avoid repeated `append` growing
3. **Use arenas (Go 1.20 experimental)** — batch allocate many short-lived objects together
4. **Avoid tiny allocations** — Go combines allocations < 16 bytes into a "tiny allocator" block; many tiny objects can still fragment
5. **Use value types in slices** — `[]Point` (contiguous) vs `[]*Point` (scattered pointers)
6. **Profile with `runtime.ReadMemStats`**:

```go
var ms runtime.MemStats
runtime.ReadMemStats(&ms)
fmt.Printf("HeapInuse: %d MB\n", ms.HeapInuse/1024/1024)
fmt.Printf("HeapIdle: %d MB\n", ms.HeapIdle/1024/1024)
fmt.Printf("HeapSys: %d MB\n", ms.HeapSys/1024/1024)
fmt.Printf("GC cycles: %d\n", ms.NumGC)
```

---

**Q26. How do goroutine stacks work in Go? What is stack growth?**

Each goroutine starts with a small stack (currently **2KB** in Go 1.22). If a goroutine needs more stack space (deep call chains), the runtime **grows the stack**:

1. Detects stack overflow during function prologue
2. Allocates a new, larger stack (double the size)
3. Copies all stack frames to the new location
4. Updates all pointers to stack variables

This is called **stack copying** (Go moved from segmented stacks to copying stacks in Go 1.4). Maximum goroutine stack is 1GB by default (configurable with `runtime.GOMAXPROCS`).

```go
// Stack growth is transparent but has cost:
func recursive(n int) int {
    if n == 0 {
        return 0
    }
    return recursive(n-1) + 1
}
// Deep recursion triggers multiple stack copies
```
