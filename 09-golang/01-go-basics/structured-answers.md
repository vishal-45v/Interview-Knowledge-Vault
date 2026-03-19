# Chapter 01 — Go Basics: Structured Answers

Full reference answers with working code examples.

---

## Answer 1: What is a slice header and how does append work?

A slice in Go is a three-field struct called the "slice header":

```go
// The runtime representation (simplified):
type SliceHeader struct {
    Data uintptr // pointer to the first element in the backing array
    Len  int     // number of accessible elements
    Cap  int     // elements from Data pointer to end of backing array
}
```

When you create a slice, you get this lightweight descriptor — not a copy of the data:

```go
s := []int{1, 2, 3, 4, 5}
// s.Data = 0xc0000b4000 (address of s[0])
// s.Len = 5
// s.Cap = 5

t := s[1:3]
// t.Data = 0xc0000b4008 (address of s[1])
// t.Len = 2
// t.Cap = 4 (from t[0] to end of backing array)
```

**How append works — two paths:**

```go
// Path 1: len < cap — no allocation, extend in place
s := make([]int, 3, 6) // len=3, cap=6
s = append(s, 10)
// No allocation. Writes 10 at index 3.
// Returns new header: Data=same, Len=4, Cap=6

// Path 2: len == cap — allocate new, larger array
s := make([]int, 3, 3)
s = append(s, 10)
// 1. New array allocated with larger cap (typically 2x for small slices)
// 2. Elements copied from old array to new
// 3. 10 written at index 3
// 4. Returns new header: Data=new_addr, Len=4, Cap=6

// CRITICAL: Always use the return value
func wrong(s []int) {
    append(s, 1)   // return value discarded — s is unchanged in caller
}

func right(s []int) []int {
    return append(s, 1) // return the new header
}
```

**Growth rate:** For slices under ~256 elements, capacity doubles. For larger slices, growth is ~1.25x to amortize allocation costs while avoiding excessive waste.

```go
// Observing growth
s := make([]int, 0)
for i := 0; i < 20; i++ {
    oldCap := cap(s)
    s = append(s, i)
    if cap(s) != oldCap {
        fmt.Printf("len=%d, new cap=%d\n", len(s), cap(s))
    }
}
// len=1, new cap=1
// len=2, new cap=2
// len=3, new cap=4
// len=5, new cap=8
// len=9, new cap=16
// len=17, new cap=32
```

---

## Answer 2: What are zero values in Go and why are they useful?

Every variable in Go is initialized to its type's zero value when declared without an explicit value. This is a language guarantee, not an implementation detail.

```go
var i int       // 0
var f float64   // 0.0
var b bool      // false
var s string    // ""
var p *int      // nil
var sl []int    // nil (but safe to use: len=0, append works)
var m map[string]int // nil (safe to read, panic on write)
var ch chan int  // nil (send/receive block forever)
var fn func()   // nil (call panics)
var err error   // nil (interface with nil type and value)
```

**Why zero values are powerful:**

1. **No uninitialized memory bugs** — C/C++ have undefined behavior from uninitialized variables; Go does not.

2. **Structs work out of the box** — if you design types well, the zero value is a valid, ready-to-use state:

```go
// sync.Mutex zero value is an unlocked mutex — ready to use
var mu sync.Mutex
mu.Lock()
// No need for sync.NewMutex()

// bytes.Buffer zero value is an empty buffer — ready to use
var buf bytes.Buffer
buf.WriteString("hello")
// No need for bytes.NewBuffer(nil)

// sync.WaitGroup zero value is ready to use
var wg sync.WaitGroup

// Custom type designed around zero value:
type RateLimiter struct {
    requests int
    limit    int  // 0 means unlimited — the zero value is a useful default
}

func (r *RateLimiter) Allow() bool {
    if r.limit == 0 {
        return true // zero value = no limit
    }
    r.requests++
    return r.requests <= r.limit
}
```

3. **Safe defaults reduce constructor complexity** — types don't always need `New()` functions.

---

## Answer 3: How do closures capture variables in Go? Show the loop closure trap.

A closure is a function that captures (closes over) variables from its lexical scope. Crucially, the closure captures the **variable itself** (a reference), not its value at the time of creation.

```go
// The closure captures the variable x, not its current value
func makeAdder(x int) func(int) int {
    return func(y int) int {
        return x + y // x is captured by reference to x's memory
    }
}

add5 := makeAdder(5)
add10 := makeAdder(10)

fmt.Println(add5(3))  // 8
fmt.Println(add10(3)) // 13
// Each call to makeAdder creates a new x on the heap
// Each returned closure has its own x
```

**The loop variable capture trap (pre-Go 1.22):**

```go
// Before Go 1.22: loop variable i is ONE variable for the entire loop
funcs := make([]func(), 5)
for i := 0; i < 5; i++ {
    funcs[i] = func() {
        fmt.Println(i) // captures i — the single loop variable
    }
}

// By the time these are called, the loop has finished and i == 5
for _, f := range funcs {
    f() // prints: 5 5 5 5 5
}
```

```go
// Fix 1: Re-declare i inside the loop body
for i := 0; i < 5; i++ {
    i := i // new variable per iteration, shadows outer i
    funcs[i] = func() { fmt.Println(i) }
}

// Fix 2: Pass as function argument
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n) // n is a parameter, not captured
    }(i)
}

// Go 1.22+ fix: loop variables are per-iteration automatically
// No special handling required
```

**Closures over mutable shared state:**

```go
func counter() (increment func(), get func() int) {
    n := 0
    increment = func() { n++ }
    get = func() int { return n }
    return
}

inc, get := counter()
inc()
inc()
inc()
fmt.Println(get()) // 3 — both closures share the same n
```

---

## Answer 4: What is defer and what order do defers execute?

`defer` schedules a function call to run when the surrounding function returns — whether by a `return` statement, reaching the end, or a `panic`. Deferred calls run in **LIFO (last in, first out)** order.

```go
func example() {
    defer fmt.Println("first defer")  // runs third
    defer fmt.Println("second defer") // runs second
    defer fmt.Println("third defer")  // runs first
    fmt.Println("function body")
}
// Output:
// function body
// third defer
// second defer
// first defer
```

**Defer is evaluated (arguments captured) immediately, but executed later:**

```go
func tricky() {
    x := 1
    defer fmt.Println("value:", x) // captures x=1 now
    x = 999
    // defer prints 1, not 999
}

func notTricky() {
    x := 1
    defer func() { fmt.Println("value:", x) }() // closure captures x (variable reference)
    x = 999
    // defer prints 999 because it reads x at execution time
}
```

**Defer for resource cleanup (canonical pattern):**

```go
func copyFile(dst, src string) (err error) {
    r, err := os.Open(src)
    if err != nil {
        return
    }
    defer r.Close()

    w, err := os.Create(dst)
    if err != nil {
        return
    }
    defer func() {
        closeErr := w.Close()
        if err == nil { // only overwrite err if no prior error
            err = closeErr
        }
    }()

    _, err = io.Copy(w, r)
    return
}
```

**Defer with panic recovery:**

```go
func safeRun(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered panic: %v", r)
        }
    }()
    fn()
    return nil
}
```

---

## Answer 5: How does Go handle multiple return values compared to exceptions?

Go's philosophy: errors are values, not exceptional events. Functions return errors as a regular last return value. The caller decides how to handle it.

```go
// Go style
func openDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("opening db: %w", err)
    }
    if err := db.Ping(); err != nil {
        db.Close()
        return nil, fmt.Errorf("pinging db: %w", err)
    }
    return db, nil
}

db, err := openDB("postgres://...")
if err != nil {
    log.Fatalf("failed to open DB: %v", err)
}
defer db.Close()
```

**Comparison with exceptions:**

| Aspect | Exceptions (Java/Python) | Go multiple returns |
|--------|--------------------------|---------------------|
| Error visibility | Hidden in call stack | Explicit in signature |
| Ignoring errors | Easy (forgotten catch) | Requires `_` — deliberate |
| Performance | Stack unwinding overhead | Zero overhead (just values) |
| Error context | Stack traces | Wrapped error messages |
| Control flow | Non-local jumps | Local — caller decides |

**Go's `panic/recover` is for truly exceptional situations:**

```go
// panic: for invariant violations, programmer errors
func mustPositive(n int) int {
    if n <= 0 {
        panic(fmt.Sprintf("expected positive, got %d", n))
    }
    return n
}

// recover: to prevent panic from crashing a server
func httpHandler(w http.ResponseWriter, r *http.Request) {
    defer func() {
        if err := recover(); err != nil {
            http.Error(w, "Internal Server Error", 500)
            log.Printf("panic: %v\n%s", err, debug.Stack())
        }
    }()
    // handle request...
}
```

---

## Answer 6: What is the difference between a pointer receiver and value receiver?

```go
type Rectangle struct {
    Width, Height float64
}

// Value receiver — receives a COPY of Rectangle
// Changes do not affect the original
// Can be called on both values and pointers
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Pointer receiver — receives a POINTER to Rectangle
// Changes DO affect the original
// Can be called on both addressable values and pointers
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

rect := Rectangle{Width: 3, Height: 4}
fmt.Println(rect.Area())   // 12 — value receiver, rect is copied
rect.Scale(2)              // equivalent to (&rect).Scale(2)
fmt.Println(rect.Area())   // 24 — pointer receiver modified rect
```

**Rules:**

1. **Use pointer receivers when** the method modifies the receiver, the struct is large (avoid copy cost), or for consistency (if any method is pointer receiver, make all pointer receivers).

2. **Use value receivers when** the method doesn't modify the receiver, the type is a small scalar-like value (e.g., `type Celsius float64`), or the type is inherently immutable.

3. **Interface satisfaction:** A pointer type `*T` has the method set of both `T` and `*T`. A non-pointer type `T` only has the method set of `T`. This means:

```go
type Stringer interface { String() string }

type MyType struct{ name string }

// If String() has a pointer receiver:
func (m *MyType) String() string { return m.name }

var s Stringer = &MyType{"hello"} // OK — *MyType has String()
// var s Stringer = MyType{"hello"} // COMPILE ERROR — MyType does not implement Stringer

// If String() has a value receiver:
func (m MyType) String() string { return m.name }

var s1 Stringer = MyType{"hello"}  // OK
var s2 Stringer = &MyType{"hello"} // Also OK — *MyType gets value methods too
```

---

## Answer 7: How do maps work internally in Go?

Go maps are implemented as hash tables with separate chaining using buckets.

```go
// A bucket holds up to 8 key-value pairs
// The map header stores:
// - pointer to array of buckets
// - number of elements
// - current capacity (number of buckets)
// - hash seed (randomized per-map creation)

m := make(map[string]int, 100) // hint: pre-allocate ~100 elements
```

**Internal structure (simplified):**
- Each key is hashed using a function seeded with a random value chosen at map creation (security against hash collision DoS attacks)
- The low-order bits of the hash select the bucket
- The high-order 8 bits (tophash) are stored in the bucket header for quick key comparison
- When a bucket overflows 8 elements, an overflow bucket is chained

**Load factor and growth:**
- When average bucket load exceeds ~6.5, the map grows (doubles bucket count)
- During growth, old and new buckets coexist; elements are incrementally migrated on reads/writes ("incremental rehashing")

```go
// Performance characteristics:
// O(1) average for Get, Set, Delete
// O(n) for iteration (visits all elements)
// Not safe for concurrent use — use sync.Map or sync.RWMutex

// Concurrent safe patterns:
var mu sync.RWMutex
data := make(map[string]int)

// Read
mu.RLock()
v := data["key"]
mu.RUnlock()

// Write
mu.Lock()
data["key"] = v + 1
mu.Unlock()

// Or use sync.Map for mostly-read, rarely-written scenarios
var sm sync.Map
sm.Store("key", 42)
v, ok := sm.Load("key")
sm.Range(func(k, v any) bool {
    fmt.Printf("%v: %v\n", k, v)
    return true // continue iteration
})
```

---

## Answer 8: What is the blank identifier `_` and when is it used?

The blank identifier `_` is a write-only variable that discards any value assigned to it. It can appear on the left side of assignments.

```go
// 1. Discard unwanted return values
result, _ := strconv.Atoi("42")         // ignore the error (usually bad practice)
_, err := fmt.Fprintf(w, "hello %s", n) // ignore bytes written

// 2. Discard loop index or value
for _, v := range items { process(v) }  // don't need index
for i := range items { process(i) }     // don't need value (range gives zero value)
for range 5 { doSomething() }            // Go 1.22+: range over integer

// 3. Import for side effects only
import _ "net/http/pprof"    // registers pprof HTTP handlers
import _ "github.com/lib/pq" // registers postgres driver

// 4. Compile-time interface assertion
type MyHandler struct{}
var _ http.Handler = (*MyHandler)(nil) // assert *MyHandler implements http.Handler
// Fails at compile time with a clear error if interface is not satisfied

// 5. Suppress "declared but not used" for variables needed for type assertion
if _, ok := i.(string); ok {
    fmt.Println("it's a string")
}

// 6. Ignore type in type switch
switch _ := v.(type) {
// ... (unusual, but valid)
}
```

---

## Answer 9: How does type assertion and type switch work?

A type assertion extracts the concrete value from an interface.

```go
var i interface{} = "hello"

// Single-value form — panics if assertion fails
s := i.(string)     // s = "hello"
n := i.(int)        // PANIC: interface conversion: interface {} is string, not int

// Comma-ok form — never panics
s, ok := i.(string) // s = "hello", ok = true
n, ok := i.(int)    // n = 0, ok = false

// Type switch — cleanly handle multiple types
func describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %q (len %d)", v, len(v))
    case bool:
        return fmt.Sprintf("bool: %t", v)
    case []int:
        return fmt.Sprintf("[]int of len %d", len(v))
    case nil:
        return "nil"
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}

// In a type switch, v has the concrete type inside each case
// In the default case, v is the interface type
```

**Practical use — JSON deserialization into interface{}:**

```go
var data interface{}
json.Unmarshal([]byte(`{"name":"Alice","age":30}`), &data)

m := data.(map[string]interface{})
name := m["name"].(string)
age := m["age"].(float64) // JSON numbers are float64 by default

// Safer with comma-ok
if age, ok := m["age"].(float64); ok {
    fmt.Printf("Age: %.0f\n", age)
}
```

---

## Answer 10: What is the difference between `var x []int` and `x := []int{}`?

```go
var x []int   // nil slice
y := []int{}  // empty (non-nil) slice

fmt.Println(x == nil)           // true
fmt.Println(y == nil)           // false

fmt.Println(len(x), cap(x))     // 0 0
fmt.Println(len(y), cap(y))     // 0 0

// Both support append:
x = append(x, 1, 2, 3) // works fine
y = append(y, 1, 2, 3) // works fine

// The difference shows up in:

// 1. JSON marshaling
jx, _ := json.Marshal(x) // null
jy, _ := json.Marshal(y) // []

// 2. reflect.DeepEqual comparison
fmt.Println(reflect.DeepEqual(x, y)) // false

// 3. Explicit nil checks in APIs
func processItems(items []string) {
    if items == nil {
        // "not provided" — different from "provided but empty"
        fmt.Println("no items provided")
        return
    }
    fmt.Printf("processing %d items\n", len(items))
}

// 4. Function returns — use nil for "no result", empty slice for "empty result"
func findUsers(query string) []User {
    if query == "" {
        return nil // no search performed
    }
    results := []User{}
    // ... query database ...
    return results // explicitly empty result set
}
```

**General rule:** Prefer `var x []int` when the zero value (nil) is meaningful (no items provided). Prefer `x := []int{}` when you need to distinguish an empty collection from a missing one, particularly for JSON APIs where `null` vs `[]` has semantic difference.
