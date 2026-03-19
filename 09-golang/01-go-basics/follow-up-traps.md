# Chapter 01 — Go Basics: Follow-Up Traps

These are counterintuitive questions designed to expose shallow understanding. Interviewers ask these after you've answered a basic question correctly to test depth.

---

## Trap 1: nil slice vs empty slice

**Q: Are `var s []int` and `s := []int{}` the same thing?**

No. They behave identically for most operations, but they are not the same:

```go
var s1 []int    // nil slice: pointer=nil, len=0, cap=0
s2 := []int{}  // empty slice: pointer=(non-nil), len=0, cap=0

fmt.Println(s1 == nil) // true
fmt.Println(s2 == nil) // false

fmt.Println(len(s1), cap(s1)) // 0 0
fmt.Println(len(s2), cap(s2)) // 0 0

// Both are safe to append to
s1 = append(s1, 1) // works fine
s2 = append(s2, 1) // works fine

// The difference matters for JSON marshaling:
import "encoding/json"
b1, _ := json.Marshal(s1) // null
b2, _ := json.Marshal(s2) // []

// It matters for reflect.DeepEqual:
fmt.Println(reflect.DeepEqual(s1, s2)) // false
```

**Key rule**: If you need to distinguish "not set" from "empty set" (e.g., JSON API response), use a nil slice for "not set" and an empty slice for "set but empty". For most internal logic, they're interchangeable.

---

## Trap 2: append and backing array aliasing

**Q: You appended to a slice and got unexpected results in another slice. What happened?**

```go
a := make([]int, 3, 6) // len=3, cap=6
b := a // b and a share the same backing array

a = append(a, 10)   // len=4, within cap — no new allocation
b = append(b, 99)   // len=4, within cap — writes to index 3 of SAME array!

fmt.Println(a) // [0 0 0 99] — NOT [0 0 0 10] !
fmt.Println(b) // [0 0 0 99]

// b's append overwrote a's element 10 because they share the array
```

Fix: use `copy` or three-index slice expression to force independence:
```go
a := make([]int, 3, 6)
b := a[:3:3] // cap=3, so append to b will allocate new array
```

**When does append guarantee a new array?** Only when `len == cap`. If there's spare capacity, the existing array is reused.

---

## Trap 3: map iteration is randomized — why?

The Go runtime deliberately randomizes map iteration order starting from Go 1.0. This was an intentional design decision to:
1. Prevent developers from accidentally depending on insertion order (which was never guaranteed and varied between Go versions)
2. Flush out bugs in code that assumed a particular order

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

// This may print in any order each run
for k, v := range m {
    fmt.Printf("%s: %d\n", k, v)
}

// The randomization seed changes each iteration start
// Even two consecutive range loops over the same map may differ
```

The randomization happens at the start of each `range` loop — Go picks a random starting bucket. This is a runtime-level randomization, not a compile-time one.

---

## Trap 4: defer in a loop doesn't execute until function returns

```go
// TRAP: This looks like it closes the file at end of each iteration
// but it doesn't — all defers stack up and run when the function returns
func readAllFiles(paths []string) error {
    for _, path := range paths {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close() // WRONG: all closes happen at the END of readAllFiles
        // process f...
    }
    return nil
}

// With 1000 files, this holds 1000 open file descriptors until return

// FIX: Extract to a function
func readOneFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // runs when readOneFile returns — correct
    // process f...
    return nil
}

// FIX 2: Use an immediately invoked function literal
for _, path := range paths {
    func() {
        f, _ := os.Open(path)
        defer f.Close() // runs when the func literal returns
        // process f...
    }()
}
```

---

## Trap 5: named return values and naked return — when is it dangerous?

```go
// Safe: short function, clear intent
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = errors.New("division by zero")
        return // naked return: returns result=0, err=...
    }
    result = a / b
    return // naked return: returns result=a/b, err=nil
}

// DANGEROUS: Long function with a naked return
func process(data []string) (results []string, err error) {
    for _, s := range data {
        // ... 50 lines of code ...
        results = append(results, transform(s))
        // ... more code ...
    }
    return // reader has no idea what's being returned here
}

// TRAP with defer and named returns — defers can modify named returns!
func alwaysPositive() (result int) {
    result = -5
    defer func() {
        if result < 0 {
            result = 0 // this DOES modify the return value!
        }
    }()
    return result // named return — defer still runs and can change it
}

fmt.Println(alwaysPositive()) // 0, not -5
```

This is one of the most common Go surprises. Named return + defer + modification = the returned value can differ from what the `return` statement specified.

---

## Trap 6: := in inner scope shadows outer variable

```go
x := 10

if true {
    x := 20 // declares a NEW x in the if-block scope
    fmt.Println(x) // 20
}

fmt.Println(x) // 10 — outer x is unchanged!

// Common bug with err:
err := doStep1()
if err != nil {
    return err
}

if true {
    result, err := doStep2() // err here is a NEW variable, not the outer err!
    _ = result
    fmt.Println(err) // inner err
}

// The outer err still holds the result from doStep1
// Fix: use = for the inner err if you want to update the outer one
var result SomeType
result, err = doStep2() // uses existing err variable
```

Go's `go vet` and linters like `staticcheck` can catch some shadowing issues but not all.

---

## Trap 7: integer overflow — no exception in Go

```go
var x int8 = 127
x++ // silently wraps to -128 — no panic, no exception!

fmt.Println(x) // -128

// Same with uint
var u uint8 = 255
u++ // wraps to 0

// No overflow detection unless you check manually or use math/big
import "math"

func safeAdd(a, b int64) (int64, error) {
    if b > 0 && a > math.MaxInt64-b {
        return 0, errors.New("integer overflow")
    }
    if b < 0 && a < math.MinInt64-b {
        return 0, errors.New("integer underflow")
    }
    return a + b, nil
}
```

This is especially dangerous in security-sensitive code (e.g., calculating offsets, buffer sizes). Unlike Java which throws `ArithmeticException` for division by zero (but not for overflow), Go silently wraps.

---

## Trap 8: string indexing gives bytes, not runes

```go
s := "Hello, 世界" // "世界" = two Chinese characters

fmt.Println(len(s))    // 13 — number of BYTES, not characters
fmt.Println(s[7])      // 228 — a byte value (uint8), not a character
fmt.Println(string(s[7])) // "ä" — garbled because 世 is 3 bytes

// Correct: use range to iterate over runes
for i, r := range s {
    fmt.Printf("index=%d rune=%c\n", i, r)
}
// index=0 rune=H
// index=7 rune=世   (byte index 7, rune index 1)
// index=10 rune=界  (byte index 10, rune index 2)

// Get rune count
runes := []rune(s)
fmt.Println(len(runes)) // 9 — H, e, l, l, o, ',', ' ', 世, 界

// Safe rune indexing
fmt.Println(string(runes[7])) // "世"
```

Strings in Go are byte sequences. The `len` builtin returns byte count. `range` over a string gives (byte-index, rune). Converting to `[]rune` gives character-level access but uses more memory.

---

## Trap 9: zero value of different types — subtle cases

```go
// These are zero values:
var i int     // 0
var f float64 // 0.0
var b bool    // false
var s string  // ""
var p *int    // nil
var sl []int  // nil
var m map[string]int // nil
var ch chan int // nil
var fn func() // nil
var iface interface{} // nil

// TRAP: nil function
var fn func() int
result := fn() // PANIC: nil function call

// TRAP: nil map read (safe) vs nil map write (panic)
var m map[string]int
_ = m["key"] // fine, returns 0
m["key"] = 1 // PANIC: assignment to entry in nil map

// TRAP: zero value of struct is usable (if fields have useful zero values)
type Config struct {
    Debug   bool   // false by default — sensible
    Timeout int    // 0 — may not be sensible for timeout!
    Name    string // "" — may not be sensible
}

// This is why some types have constructors like New() to set sensible defaults
```

---

## Trap 10: comparing structs with == when a field is incomparable

```go
type Good struct {
    Name string
    Age  int
}

type Bad struct {
    Name string
    Tags []string // slice is not comparable
}

g1 := Good{"Alice", 30}
g2 := Good{"Alice", 30}
fmt.Println(g1 == g2) // true — compiles and works

b1 := Bad{"Alice", []string{"admin"}}
b2 := Bad{"Alice", []string{"admin"}}
// fmt.Println(b1 == b2) // COMPILE ERROR: invalid operation: b1 == b2

// To compare structs with incomparable fields, use reflect.DeepEqual
fmt.Println(reflect.DeepEqual(b1, b2)) // true (but slower)

// Or implement your own Equal method
func (b Bad) Equal(other Bad) bool {
    if b.Name != other.Name { return false }
    if len(b.Tags) != len(other.Tags) { return false }
    for i := range b.Tags {
        if b.Tags[i] != other.Tags[i] { return false }
    }
    return true
}
```

---

## Trap 11: defer evaluates arguments immediately but executes body later

```go
func trap() {
    x := 10
    defer fmt.Println("deferred x:", x) // x is evaluated NOW (captures 10)
    x = 20
    fmt.Println("current x:", x)
}
// Output:
// current x: 20
// deferred x: 10   ← NOT 20, because x was evaluated at the defer line

// To capture the current value at execution time, use a closure:
func notATrap() {
    x := 10
    defer func() {
        fmt.Println("deferred x:", x) // x evaluated when func runs
    }()
    x = 20
    fmt.Println("current x:", x)
}
// Output:
// current x: 20
// deferred x: 20   ← closure captures the variable, not its value
```

---

## Trap 12: blank identifier to satisfy interface — compile-time check

```go
type Writer interface {
    Write([]byte) (int, error)
}

type MyWriter struct{}

// This does NOT compile if MyWriter doesn't implement Writer
// It's a deliberate compile-time assertion
var _ Writer = (*MyWriter)(nil)

// Use this pattern in packages to catch interface drift early
// When someone removes a method from MyWriter, the compilation fails here
// with a clear error rather than failing at a distant usage site
```

This is different from a test assertion — it's a zero-cost compile-time guarantee. Commonly placed near the type definition.

---

## Trap 13: range copies the value

```go
type Point struct{ X, Y int }

points := []Point{{1, 2}, {3, 4}, {5, 6}}

// TRAP: v is a copy, modifying it doesn't affect the slice
for _, v := range points {
    v.X *= 2 // modifies the copy, not the original
}
fmt.Println(points) // [{1 2} {3 4} {5 6}] — unchanged!

// FIX: Use index
for i := range points {
    points[i].X *= 2 // modifies the original
}

// Or use pointer slice
pPoints := []*Point{{1, 2}, {3, 4}}
for _, p := range pPoints {
    p.X *= 2 // p is a pointer copy, but *p modifies the original
}
```
