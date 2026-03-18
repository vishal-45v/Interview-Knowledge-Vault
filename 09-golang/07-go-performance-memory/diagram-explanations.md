# Go Performance & Memory — Diagram Explanations

---

## Diagram 1: GC Tri-Color Marking

```
INITIAL STATE: All objects are WHITE (unvisited)

   [A] ---> [B] ---> [D]
    |
    +------> [C] ---> [E]

All objects: WHITE (A, B, C, D, E)
Roots: A is a root (stack variable or global)

─────────────────────────────────────────────────────────
STEP 1: Mark roots GREY

   [A]* ---> [B] ---> [D]
    |
    +-------> [C] ---> [E]

   * = grey (discovered, children not yet scanned)
   GREY queue: {A}
   WHITE: B, C, D, E

─────────────────────────────────────────────────────────
STEP 2: Process A — scan children, mark them GREY, mark A BLACK

   [A]■ ---> [B]* ---> [D]
    |
    +-------> [C]* ---> [E]

   ■ = black (fully scanned)
   * = grey
   BLACK: A
   GREY queue: {B, C}
   WHITE: D, E

─────────────────────────────────────────────────────────
STEP 3: Process B — mark D grey, mark B black

   [A]■ ---> [B]■ ---> [D]*
    |
    +-------> [C]* ---> [E]

   BLACK: A, B
   GREY queue: {C, D}
   WHITE: E

─────────────────────────────────────────────────────────
STEP 4: Process C — mark E grey, mark C black

   [A]■ ---> [B]■ ---> [D]*
    |
    +-------> [C]■ ---> [E]*

   BLACK: A, B, C
   GREY queue: {D, E}

─────────────────────────────────────────────────────────
STEP 5: Process D and E — no children, mark black

   [A]■ ---> [B]■ ---> [D]■
    |
    +-------> [C]■ ---> [E]■

   BLACK: A, B, C, D, E
   GREY queue: {} (empty!)
   WHITE: (none — all reachable objects are black)

─────────────────────────────────────────────────────────
STEP 6: Sweep — collect all remaining WHITE objects

   In this example: no WHITE objects = no memory freed.

   If object [F] existed with NO references:
   [F] stays WHITE → collected by sweep → memory returned to heap.

COLOR KEY:
   □ = WHITE  — not yet visited / unreachable (candidate for collection)
   *  = GREY  — discovered but children not yet scanned
   ■ = BLACK  — fully scanned, all references followed

TRI-COLOR INVARIANT:
   A BLACK object must NEVER directly point to a WHITE object.
   The WRITE BARRIER enforces this during concurrent marking.
```

---

## Diagram 2: Stack vs Heap — Escape Analysis Decision Tree

```
         Variable declared in a function
                      │
                      ▼
          ┌─────────────────────┐
          │ Does its address     │
          │ escape the function? │
          └─────────────────────┘
                   │
           ┌───────┴───────┐
          YES              NO
           │               │
           ▼               ▼
  ┌──────────────┐   ┌──────────────┐
  │ Returned as  │   │ Returned by  │
  │ pointer?     │   │ value (copy) │
  └──────────────┘   └──────────────┘
         YES               │
          │            ┌───┴─────────────┐
          ▼            │ Used only within │
    ┌──────────┐       │ this function?  │
    │ HEAP     │       └─────────────────┘
    │ ALLOC    │                YES
    └──────────┘                 │
         │                       ▼
  Other escape             ┌──────────┐
  causes → HEAP:           │ STACK    │
  • Stored in interface{}  │ ALLOC    │
  • Goroutine closure      └──────────┘
  • Stored in escaping          │
    data structure         Fast! No GC.
  • Size > 32KB            Auto-freed on
                           function return.

COMPILE-TIME CHECK:
   go build -gcflags="-m" ./...

   Output examples:
   "./main.go:8:2: x escapes to heap"
   "./main.go:14:2: x does not escape"

PERFORMANCE COMPARISON:
   Stack alloc:  ~0 ns (just move the stack pointer)
   Heap alloc:   ~20-50 ns (runtime.mallocgc + potential GC scan)

   At 1M calls/sec:
   Stack: 0 ms GC overhead
   Heap:  ~50ms allocation + GC pressure
```

---

## Diagram 3: sync.Pool Lifecycle

```
                        ┌─────────────────┐
                        │   sync.Pool{}   │
                        │  New: func()    │
                        │  local[P] cache │
                        │  victim cache   │
                        └────────┬────────┘
                                 │
          ┌──────────────────────┴──────────────────────┐
          │                                              │
          ▼                                              ▼
  pool.Get() called                             pool.Put(obj) called
          │                                              │
          ▼                                              ▼
  ┌───────────────────┐                    ┌────────────────────────┐
  │ Check local cache │                    │ Place in local[P] cache│
  │ for this P        │                    │ (per-CPU, no lock)     │
  └────────┬──────────┘                    └────────────────────────┘
           │
   ┌───────┴───────┐
  FOUND           NOT FOUND
   │               │
   ▼               ▼
Return obj    Check victim cache
(reused!)          │
              ┌────┴────┐
           FOUND     NOT FOUND
              │         │
              ▼         ▼
          Return    Call New()
          (reused!) (fresh alloc)

─────────────── GC CYCLE ───────────────

  BEFORE GC:
  ┌─────────────┬──────────────┐
  │ local cache │ victim cache │
  │  [obj1]     │  [obj3]      │
  │  [obj2]     │  [obj4]      │
  └─────────────┴──────────────┘

  DURING GC (STW):
  victim cache ──► EVICTED (collected by GC)
  local cache  ──► becomes new victim cache

  AFTER GC:
  ┌─────────────┬──────────────┐
  │ local cache │ victim cache │
  │  (empty)    │  [obj1]      │
  │             │  [obj2]      │
  └─────────────┴──────────────┘

  After SECOND GC: obj1, obj2 also evicted.

KEY INSIGHT: Two GC cycles needed to fully clear a pool.
(This is why runtime.GC() once in tests doesn't reliably clear the pool.)

CORRECT USAGE PATTERN:
  obj := pool.Get().(*MyType)
  obj.Reset()           // ← always reset before use
  defer pool.Put(obj)   // ← always return (even on error/panic)
  // ... use obj ...
```

---

## Diagram 4: pprof Profiling Stack (Flame Graph Concept)

```
Reading a CPU Flame Graph:

 WIDTH = CPU time (or memory) consumed
 HEIGHT = call stack depth
 BASE = entry points (main, goroutine roots)
 TOP = leaf functions (where execution actually happens)

          ┌──────────────────────────────────────────┐
          │              json.Unmarshal               │  ← HOT SPOT (wide = slow)
          │   (35% of total CPU time)                  │
     ┌────┴───────────┐ ┌────────────────────────────┐
     │ json.(*decoder) │ │   json.indirect            │
     │ .decode()       │ │   (reflection overhead)    │
  ┌──┴────────────┐   │ └────────────────────────────┘
  │ reflect.Value │   │
  │ .Set()        │   │
  └───────────────┘   │
          ┌───────────┴──────────────────────────────┐
          │           handleRequest()                  │
          │           (45% total — wide but not flat) │
          ├────────────────────────────────────────────┤
          │             http.(*ServeMux)               │
          │             .ServeHTTP()                   │
          ├────────────────────────────────────────────┤
          │          http.(*Server).Serve()            │
          └────────────────────────────────────────────┘

HOW TO READ:
  1. Find the WIDEST flat top — that's your bottleneck
  2. Follow it DOWN to find which caller is responsible
  3. The bottom of the flame = goroutine entry/main

GENERATE THE FLAME GRAPH:
  go tool pprof -http=:8080 cpu.prof
  → Open browser → click "Flame Graph"

PPROF PROFILE TYPES:
  ┌────────────────┬──────────────────────────────────────┐
  │ Profile        │ What it shows                        │
  ├────────────────┼──────────────────────────────────────┤
  │ cpu            │ Where goroutines spend CPU time       │
  │ heap           │ Current live allocations              │
  │ allocs         │ Total allocations (including freed)   │
  │ goroutine      │ Stack traces of all goroutines        │
  │ block          │ Where goroutines blocked (chan/mutex)  │
  │ mutex          │ Mutex contention hot spots            │
  │ threadcreate   │ OS thread creation events             │
  └────────────────┴──────────────────────────────────────┘
```

---

## Diagram 5: Struct Field Padding — Bad vs Good Layout

```
BAD LAYOUT (fields in declaration order — compiler inserts padding):

type Wasteful struct {
    A bool      // 1 byte
    B float64   // 8 bytes  (needs 8-byte alignment)
    C int32     // 4 bytes  (needs 4-byte alignment)
    D byte      // 1 byte
}

Memory layout:
Offset: 0    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15   16   17   18   19   20   21   22   23
        ┌────┬────┴────┴────┴────┴────┴────┴────┼────┴────┴────┴────┼────┴────┴────┴────┴────┴────┴────┴────┼────┬────
        │ A  │  padding (7 bytes wasted)         │    B (8 bytes)    │    C (4 bytes)   │ D  │padding(3 bytes)│
        └────┴─────────────────────────────────────────────────────────────────────────────────────────────────────

TOTAL: 24 bytes  (10 bytes wasted = 42% waste!)


GOOD LAYOUT (sort fields largest → smallest):

type Efficient struct {
    B float64   // 8 bytes  (offset 0, aligned ✓)
    C int32     // 4 bytes  (offset 8, aligned ✓)
    A bool      // 1 byte   (offset 12)
    D byte      // 1 byte   (offset 13)
              // 2 bytes padding at end (struct must align to 8)
}

Memory layout:
Offset: 0    1    2    3    4    5    6    7    8    9   10   11   12   13   14   15
        ┌────┴────┴────┴────┴────┴────┴────┴────┼────┴────┴────┴────┼────┼────┼────┴────
        │         B (8 bytes, float64)           │ C (4 bytes,int32) │ A  │ D  │pad(2)  │
        └─────────────────────────────────────────────────────────────────────────────────

TOTAL: 16 bytes  (2 bytes wasted = 12.5% waste)
SAVINGS: 8 bytes per struct = 33% smaller!

AT SCALE:
  10 million structs × 8 bytes saved = 80 MB freed

ALIGNMENT RULES (field needs N-byte aligned offset):
  ┌────────────┬────────────────┬──────────────────┐
  │ Type       │ Size (bytes)   │ Alignment        │
  ├────────────┼────────────────┼──────────────────┤
  │ bool, byte │ 1              │ 1 (any offset)   │
  │ int16      │ 2              │ 2-byte boundary  │
  │ int32,f32  │ 4              │ 4-byte boundary  │
  │ int64,f64  │ 8              │ 8-byte boundary  │
  │ pointer    │ 8 (64-bit)     │ 8-byte boundary  │
  │ struct     │ sum+padding    │ max field align  │
  └────────────┴────────────────┴──────────────────┘

TOOL: fieldalignment ./...  (golang.org/x/tools)
CODE: unsafe.Sizeof(MyStruct{})  — check current size
```

---

## Diagram 6: Slice Sub-Slice Memory Leak

```
INITIAL ALLOCATION:

var data = make([]byte, 1_000_000)  // 1 MB buffer
// data is filled with log file content

data:
  ┌──────────────────────────────────────────────────────────────────┐
  │ ptr ──────────────────────────────────────────────────────────► │
  │ len: 1,000,000                                                   │
  │ cap: 1,000,000                                                   │
  └──────────────────────────────────────────────────────────────────┘
             │
             ▼
  ┌──── BACKING ARRAY (1 MB) ─────────────────────────────────────────
  │ ...log line...│req_id=abc123 │...more log data...                 │
  └────────────────────────────────────────────────────────────────────
      offset 0                                          offset 999,999

─────────────────────────────────────────────────────────────────────────

THE LEAK (sub-slicing without copying):

header := data[100:120]  // "req_id=abc123" — only 20 bytes

header:
  ┌──────────────────────────────────────────────────────────────────┐
  │ ptr ──────────── points INTO the same backing array!             │
  │ len: 20                                                          │
  │ cap: 999,900  (remaining capacity of original!)                  │
  └──────────────────────────────────────────────────────────────────┘
             │
             ▼
  ┌──── BACKING ARRAY (1 MB) — STILL ALIVE! ─────────────────────────
  │ ...log line...│req_id=abc123 │...more log data...                 │
  └────────────────────────────────────────────────────────────────────
                   ^
                   │  header points here

GC SEES: header is alive → backing array is alive → 1 MB retained!

Even though `data` goes out of scope and becomes unreachable,
`header` keeps the entire 1 MB backing array alive.

─────────────────────────────────────────────────────────────────────────

THE FIX (copy to break the reference):

result := make([]byte, 20)   // new 20-byte allocation
copy(result, data[100:120])  // copy only the needed bytes

result:
  ┌──────────────────────────────────────────────────────────────────┐
  │ ptr ──── points to its own backing array                         │
  │ len: 20                                                          │
  │ cap: 20                                                          │
  └──────────────────────────────────────────────────────────────────┘
             │
             ▼
  ┌── NEW BACKING ARRAY (20 bytes) ──┐
  │ req_id=abc123                    │
  └──────────────────────────────────┘

Now `data`'s 1 MB backing array can be GC'd. Only 20 bytes retained.

─────────────────────────────────────────────────────────────────────────

QUICK FIX ONE-LINER:
  result := append([]byte(nil), data[100:120]...)
  // append creates a new backing array with only len=cap=20

SAME PATTERN EXISTS WITH STRINGS:
  bigResponse := fetchHTTP()             // 500KB HTTP response body
  contentType := bigResponse[100:130]    // retains entire 500KB!

  // Fix:
  contentType = string([]byte(bigResponse[100:130]))  // force copy
```
