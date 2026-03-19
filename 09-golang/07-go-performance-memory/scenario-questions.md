# Go Performance & Memory — Scenario Questions

---

## Scenario 1: High Memory Usage Investigation

**"Your Go service has been running for 6 hours. Memory is at 4GB and climbing. Walk through exactly how you'd investigate it."**

**Step 1 — Establish a baseline with live heap profile:**
```bash
# Assuming pprof endpoint is enabled
curl http://localhost:6060/debug/pprof/heap > heap1.prof
# Wait 30 minutes
curl http://localhost:6060/debug/pprof/heap > heap2.prof
go tool pprof -http=:8080 heap2.prof
```

**Step 2 — Check if it's a leak or normal growth:**
```go
// Add runtime metrics endpoint
http.HandleFunc("/metrics/mem", func(w http.ResponseWriter, r *http.Request) {
    var ms runtime.MemStats
    runtime.ReadMemStats(&ms)
    fmt.Fprintf(w, "HeapAlloc=%dMB HeapInuse=%dMB HeapIdle=%dMB NumGC=%d\n",
        ms.HeapAlloc>>20, ms.HeapInuse>>20, ms.HeapIdle>>20, ms.NumGC)
})
```

If `HeapInuse` grows while `HeapIdle` is low → live objects growing (leak).
If `HeapIdle` is large → memory returned to OS is being re-requested.

**Step 3 — Look at inuse_space vs alloc_space:**
```bash
go tool pprof -inuse_space heap2.prof  # what is alive now
go tool pprof -alloc_space heap2.prof  # what allocated the most total
```

**Step 4 — Check goroutine count:**
```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=1 | head -50
```
Thousands of goroutines = goroutine leak.

**Step 5 — Common culprits to check:**
- Unbounded in-memory caches with no eviction
- Goroutine leak (channels never closed)
- Accumulation of request context values
- `time.After` inside a loop (each creates a timer that holds memory until it fires)
- Sub-slice retaining large backing array

---

## Scenario 2: Millions of Small Allocations Per Second

**"A function is creating millions of small allocations per second, causing GC to run constantly. The CPU profile shows 30% time in runtime.mallocgc. How do you fix it?"**

**Diagnosis:**
```bash
go test -bench=BenchmarkProcess -benchmem -cpuprofile=cpu.prof
go tool pprof cpu.prof
(pprof) top20
# See: runtime.mallocgc, runtime.memmove high on list
```

**Fix 1 — Use sync.Pool for reusable objects:**
```go
// Before: allocates a new Buffer every call
func processRequest(data []byte) string {
    buf := &bytes.Buffer{}
    buf.Write(data)
    // transform...
    return buf.String()
}

// After: reuse buffers
var bufPool = sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}

func processRequest(data []byte) string {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)
    buf.Write(data)
    return buf.String()
}
```

**Fix 2 — Pre-allocate slices:**
```go
// Before: repeated allocations
func collectResults(n int) []Result {
    var results []Result
    for i := 0; i < n; i++ {
        results = append(results, computeResult(i))
    }
    return results
}

// After: single allocation
func collectResults(n int) []Result {
    results := make([]Result, 0, n)
    for i := 0; i < n; i++ {
        results = append(results, computeResult(i))
    }
    return results
}
```

**Fix 3 — Avoid interface boxing in hot path:**
```go
// Before: boxes int into interface{} on every call
func logValue(v interface{}) {
    // ...
}
logValue(42) // allocation

// After: use concrete type or accept the specific type
func logInt(v int) {
    // no allocation
}
```

**Fix 4 — String building:**
```go
// Before: O(n^2) allocations
func buildCSV(rows [][]string) string {
    result := ""
    for _, row := range rows {
        result += strings.Join(row, ",") + "\n"
    }
    return result
}

// After:
func buildCSV(rows [][]string) string {
    var sb strings.Builder
    for _, row := range rows {
        sb.WriteString(strings.Join(row, ","))
        sb.WriteByte('\n')
    }
    return sb.String()
}
```

---

## Scenario 3: CPU Bottleneck in HTTP Server

**"You have a Go HTTP server handling 10,000 req/s. p99 latency is 200ms. Use pprof to find the bottleneck."**

**Step 1 — Expose pprof and collect under load:**
```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe(":6060", nil) // pprof port
    http.ListenAndServe(":8080", handler) // app port
}
```

```bash
# While load test is running:
curl "http://localhost:6060/debug/pprof/profile?seconds=30" > cpu.prof
go tool pprof -http=:8081 cpu.prof
```

**Step 2 — Interpret the flame graph:**
- Wide flat tops = where CPU time is spent
- Look for unexpected functions dominating

**Step 3 — Common findings and fixes:**

```go
// Finding: json.Unmarshal in hot path
// Fix: use sonic or easyjson for hot paths
import "github.com/bytedance/sonic"

var result MyStruct
sonic.Unmarshal(body, &result) // 3-5x faster

// Finding: regexp.Compile called on every request
// Fix: compile once at package level
var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$`)

func validateEmail(s string) bool {
    return emailRegex.MatchString(s)
}

// Finding: database connection pool exhausted (goroutines waiting)
// Use trace instead of pprof to see blocking:
curl "http://localhost:6060/debug/pprof/trace?seconds=5" > trace.out
go tool trace trace.out
```

**Step 4 — Goroutine profile for I/O-bound issues:**
```bash
curl "http://localhost:6060/debug/pprof/goroutine?debug=2" > goroutine.txt
# Look for goroutines stuck on: chan receive, net.(*netFD).Read, etc.
```

---

## Scenario 4: High-Throughput JSON Processing Service

**"You're building a service that processes 50,000 JSON messages/second from Kafka. What optimizations matter most?"**

**Optimization 1 — Reuse decoder/encoder:**
```go
// Bad: new decoder per message
func process(msg []byte) (*Event, error) {
    var e Event
    return &e, json.Unmarshal(msg, &e)
}

// Better: use sonic (SIMD-accelerated JSON)
import "github.com/bytedance/sonic"

func process(msg []byte) (*Event, error) {
    var e Event
    return &e, sonic.Unmarshal(msg, &e)
}
```

**Optimization 2 — Pool the Event objects:**
```go
var eventPool = sync.Pool{
    New: func() interface{} { return new(Event) },
}

func processMessage(msg []byte) error {
    e := eventPool.Get().(*Event)
    defer func() {
        *e = Event{} // zero it before returning
        eventPool.Put(e)
    }()

    if err := sonic.Unmarshal(msg, e); err != nil {
        return err
    }
    return handleEvent(e)
}
```

**Optimization 3 — Avoid allocation in field extraction:**
```go
// Bad: allocates a new string for each field
type Event struct {
    UserID string `json:"user_id"`
    Action string `json:"action"`
}

// Better for read-only: use json.RawMessage or gjson for partial parsing
import "github.com/tidwall/gjson"

func getAction(msg []byte) string {
    return gjson.GetBytes(msg, "action").String() // zero alloc for simple reads
}
```

**Optimization 4 — Batch processing:**
```go
func processBatch(msgs [][]byte) []error {
    errs := make([]error, len(msgs))
    // Process in parallel with a worker pool
    var wg sync.WaitGroup
    sem := make(chan struct{}, runtime.NumCPU())
    for i, msg := range msgs {
        wg.Add(1)
        sem <- struct{}{}
        go func(i int, msg []byte) {
            defer wg.Done()
            defer func() { <-sem }()
            errs[i] = processMessage(msg)
        }(i, msg)
    }
    wg.Wait()
    return errs
}
```

**Optimization 5 — Struct layout for hot data:**
```go
// Align frequently accessed fields together (cache line = 64 bytes)
type Event struct {
    // Hot fields first (read every message):
    Timestamp int64   // 8
    UserID    uint64  // 8
    Type      uint8   // 1
    _         [7]byte // explicit padding for alignment

    // Cold fields (only needed sometimes):
    Metadata  map[string]string
    Tags      []string
}
```

---

## Scenario 5: Investigating Escape Analysis

**"A benchmark shows your function is 3x slower than expected. You suspect heap allocations. Walk through using escape analysis to diagnose it."**

```go
// Original code — mysteriously slow
func sumSquares(nums []float64) float64 {
    var total float64
    for _, n := range nums {
        sq := &Square{Value: n * n}
        total += sq.Value
    }
    return total
}

type Square struct{ Value float64 }
```

**Step 1 — Run escape analysis:**
```bash
go build -gcflags="-m=1" .
# Output: sq escapes to heap
```

**Step 2 — Benchmark to confirm:**
```go
func BenchmarkSumSquares(b *testing.B) {
    nums := make([]float64, 1000)
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        sumSquares(nums)
    }
}
// Output: 1000 allocs/op — one per loop iteration!
```

**Step 3 — Fix: avoid unnecessary pointer:**
```go
// Fixed: no pointer, Square stays on stack
func sumSquares(nums []float64) float64 {
    var total float64
    for _, n := range nums {
        sq := Square{Value: n * n}  // value type, no escape
        total += sq.Value
    }
    return total
}
// Or just: total += n * n  (no struct needed at all)
```

**Step 4 — Verify fix:**
```bash
go build -gcflags="-m=1" .
# No escape message for sq
go test -bench=BenchmarkSumSquares -benchmem
# Output: 0 allocs/op
```

---

## Scenario 6: sync.Pool in a Web Server

**"Your web server parses large request bodies into a struct. Under load, GC is spending 15% of CPU. Implement sync.Pool to fix it."**

```go
type RequestParser struct {
    buf     []byte
    decoder *json.Decoder
}

var parserPool = sync.Pool{
    New: func() interface{} {
        return &RequestParser{
            buf: make([]byte, 0, 4096),
        }
    },
}

func handleRequest(w http.ResponseWriter, r *http.Request) {
    parser := parserPool.Get().(*RequestParser)
    defer parserPool.Put(parser)

    // Reset state before use
    parser.buf = parser.buf[:0]

    body, err := io.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "bad request", 400)
        return
    }

    var req MyRequest
    if err := json.Unmarshal(body, &req); err != nil {
        http.Error(w, "invalid JSON", 400)
        return
    }

    processAndRespond(w, &req)
}
```

Key rules when using sync.Pool:
1. Always `Reset()` or zero the object before putting back
2. Never put objects that reference request-scoped data
3. Never assume the object you get back is clean — always reset first
4. Never put back objects in a failed state

---

## Scenario 7: Reducing Struct Padding

**"A profiler shows you have 10 million instances of a struct in memory. The service uses 800MB. You need to halve that."**

```go
// Original struct — bad alignment
type OriginalRecord struct {
    Active    bool      // 1 byte
    _         [7]byte   // implicit padding
    Score     float64   // 8 bytes
    Count     int32     // 4 bytes
    _         [4]byte   // implicit padding
    Category  byte      // 1 byte
    _         [7]byte   // implicit padding
    Timestamp int64     // 8 bytes
}
// Total: 40 bytes × 10M = 400MB — but let's check actual
```

```go
import "unsafe"
fmt.Println(unsafe.Sizeof(OriginalRecord{})) // 40
```

```go
// Optimized: sort fields largest → smallest
type OptimizedRecord struct {
    Score     float64 // 8 bytes (offset 0)
    Timestamp int64   // 8 bytes (offset 8)
    Count     int32   // 4 bytes (offset 16)
    Category  byte    // 1 byte  (offset 20)
    Active    bool    // 1 byte  (offset 21)
    _         [2]byte // 2 bytes explicit padding (align to 8)
}
// Total: 24 bytes × 10M = 240MB — saves 160MB!
```

Verify:
```bash
go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest
fieldalignment ./...
```

---

## Scenario 8: Memory Leak via Sub-Slice

**"A service that processes large byte slices is slowly leaking memory. You suspect sub-slice retention. Diagnose and fix it."**

```go
// Bug: retaining entire 1MB log line when only 20 bytes are needed
func extractRequestID(logLine []byte) []byte {
    start := bytes.Index(logLine, []byte("req_id="))
    if start == -1 {
        return nil
    }
    start += len("req_id=")
    end := bytes.IndexByte(logLine[start:], ' ')
    if end == -1 {
        return logLine[start:] // retains entire logLine backing array!
    }
    return logLine[start : start+end] // same problem
}

// Fix: copy the relevant bytes into a new slice
func extractRequestID(logLine []byte) []byte {
    start := bytes.Index(logLine, []byte("req_id="))
    if start == -1 {
        return nil
    }
    start += len("req_id=")
    end := bytes.IndexByte(logLine[start:], ' ')

    var id []byte
    if end == -1 {
        id = logLine[start:]
    } else {
        id = logLine[start : start+end]
    }

    // Copy to break the reference to the original large slice
    result := make([]byte, len(id))
    copy(result, id)
    return result
}
```

The same pattern applies to strings:
```go
// String interning / retention bug
func cacheHeader(resp *http.Response) {
    // resp.Header["Content-Type"][0] is a sub-string of the raw HTTP response
    // which is a large []byte. Caching it retains the whole response in memory.
    contentType := resp.Header.Get("Content-Type")

    // Fix: force a copy
    contentType = string([]byte(contentType))
    cache.Set("content-type", contentType)
}
```

---

## Scenario 9: Profiling a Long-Running Daemon

**"You have a Go daemon that processes jobs from a queue. After 24 hours, it consumes twice as much CPU as at startup. How do you investigate?"**

```go
// 1. Take baseline profile at startup
// 2. Take profile after 24 hours
// 3. Compare with benchstat or diff profiles

// Setup continuous profiling endpoint:
import (
    "net/http"
    _ "net/http/pprof"
    "runtime"
    "time"
)

func startProfilingServer() {
    go http.ListenAndServe(":6060", nil)
}

// Collect profiles at intervals:
func periodicProfile(interval time.Duration) {
    ticker := time.NewTicker(interval)
    for range ticker.C {
        f, _ := os.Create(fmt.Sprintf("heap_%d.prof", time.Now().Unix()))
        runtime.GC()
        pprof.WriteHeapProfile(f)
        f.Close()
    }
}
```

**Diff two profiles:**
```bash
go tool pprof -base=heap_t0.prof heap_t24.prof
(pprof) top10 -cum  # see what grew
```

**Common causes of CPU increase over time:**
- Growing caches → more GC pressure → more GC CPU
- Goroutine count growing → more scheduler overhead
- Hot paths that allocate frequently (timer.After in loops)
- Connection pool thrashing

---

## Scenario 10: Optimizing a Sorting Hot Path

**"A report generation service sorts a 100,000-element slice on every request. It's the top CPU consumer. What are your options?"**

```go
// Baseline measurement
func BenchmarkSort(b *testing.B) {
    data := generateData(100000)
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        sort.Slice(data, func(i, j int) bool {
            return data[i].Score > data[j].Score
        })
    }
}

// Option 1: Use sort.Sort with a concrete type (avoids interface overhead)
type ByScore []Record
func (s ByScore) Len() int           { return len(s) }
func (s ByScore) Less(i, j int) bool { return s[i].Score > s[j].Score }
func (s ByScore) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

// Option 2: Use slices.SortFunc (Go 1.21) — avoids reflect
import "slices"
slices.SortFunc(data, func(a, b Record) int {
    return cmp.Compare(b.Score, a.Score)
})

// Option 3: Pre-sort or maintain a sorted structure (heap/treap)
// if data changes incrementally

// Option 4: Radix sort if keys are integers/fixed-size
```

Benchmark comparison reveals `slices.SortFunc` is typically 20–30% faster than `sort.Slice` due to generics avoiding interface boxing.
