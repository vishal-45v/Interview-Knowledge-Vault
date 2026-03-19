# Chapter 02 — Go OOP & Interfaces: Follow-Up Traps

---

## Trap 1: nil interface is NOT nil when it holds a nil pointer of a concrete type

This is Go's most notorious trap, appearing in real-world code constantly.

```go
type MyError struct{ code int }
func (e *MyError) Error() string { return fmt.Sprintf("error code %d", e.code) }

func checkThings(ok bool) error {
    var err *MyError // nil pointer of concrete type
    if !ok {
        err = &MyError{code: 42}
    }
    return err // ALWAYS returns a non-nil interface! Even when ok=true
}

err := checkThings(true)
fmt.Println(err == nil) // false — SURPRISE!
fmt.Println(err)        // <nil>
// err.Error() would PANIC — the interface is non-nil but holds a nil pointer

// Why: the interface has TWO components:
// interface{ type: *MyError, value: nil } — the type part is non-nil!
// Only interface{ type: nil, value: nil } is truly nil

// Fix: return untyped nil
func checkThingsFixed(ok bool) error {
    if !ok {
        return &MyError{code: 42}
    }
    return nil // truly nil interface
}
```

Rule: **Never return a typed nil as an interface. Return untyped `nil` directly.**

---

## Trap 2: Value receiver vs pointer receiver — which satisfies the interface?

```go
type Printer interface {
    Print()
}

type Doc struct{ content string }

// Case A: value receiver
func (d Doc) Print() { fmt.Println(d.content) }

var p Printer = Doc{content: "hello"}     // OK — Doc has Print()
var p Printer = &Doc{content: "hello"}    // OK — *Doc also has value methods

// Case B: pointer receiver
func (d *Doc) Print() { fmt.Println(d.content) }

var p Printer = &Doc{content: "hello"}   // OK — *Doc has Print()
var p Printer = Doc{content: "hello"}    // COMPILE ERROR!
// Cannot use Doc value as Printer — only *Doc satisfies Printer

// Why the asymmetry?
// Go can take the address of an addressable value (&Doc{}),
// so *Doc can use both pointer and value methods.
// But Go cannot automatically dereference a non-pointer,
// especially if the value isn't addressable (e.g., map elements, return values).

m := map[string]Doc{"key": {content: "hi"}}
// m["key"].Print() — if Print() has pointer receiver, this is a COMPILE ERROR
// because map values are not addressable
```

**Memory rule:** `T`'s method set = value receivers of T. `*T`'s method set = value receivers of T + pointer receivers of T.

---

## Trap 3: Struct embedding — method ambiguity when both types have the same method

```go
type Logger struct{}
func (Logger) Log(msg string) { fmt.Println("Logger:", msg) }

type Auditor struct{}
func (Auditor) Log(msg string) { fmt.Println("Auditor:", msg) }

type Service struct {
    Logger
    Auditor
}

s := Service{}

// s.Log("hi") — COMPILE ERROR: ambiguous selector s.Log
// Both Logger.Log and Auditor.Log are at the same depth

// Must qualify:
s.Logger.Log("hi")  // OK
s.Auditor.Log("hi") // OK

// If Service defines its own Log, it takes precedence over both:
func (s Service) Log(msg string) {
    s.Logger.Log(msg)
    s.Auditor.Log(msg)
}
s.Log("hi") // Now unambiguous — uses Service.Log
```

Depth matters: if one embedded type is at a shallower depth than another, the shallower one wins (no ambiguity error).

---

## Trap 4: Comparing interfaces — runtime panic for non-comparable underlying type

```go
var a, b interface{}

// Safe comparisons:
a, b = 42, 42
fmt.Println(a == b) // true

a, b = "hello", "hello"
fmt.Println(a == b) // true

// RUNTIME PANIC with slices:
a, b = []int{1, 2, 3}, []int{1, 2, 3}
fmt.Println(a == b) // PANIC: runtime error: comparing uncomparable type []int

// RUNTIME PANIC with maps:
a, b = map[string]int{"a": 1}, map[string]int{"a": 1}
fmt.Println(a == b) // PANIC

// Safe approach: use reflect.DeepEqual for deep comparison
fmt.Println(reflect.DeepEqual(a, b)) // true — but slower, uses reflection

// Or type-assert and compare concretely:
if sa, ok := a.([]int); ok {
    if sb, ok := b.([]int); ok {
        fmt.Println(slices.Equal(sa, sb)) // Go 1.21+ slices package
    }
}
```

---

## Trap 5: Empty interface{} — performance and type safety implications vs generics

```go
// Pre-generics: use interface{}/any for generic containers
type Stack struct {
    items []interface{}
}

func (s *Stack) Push(v interface{}) { s.items = append(s.items, v) }
func (s *Stack) Pop() interface{}   {
    n := len(s.items)
    v := s.items[n-1]
    s.items = s.items[:n-1]
    return v
}

// Problems:
s := &Stack{}
s.Push(42)
s.Push("hello") // No type safety — mixing types is allowed
v := s.Pop()
n := v.(int) // Type assertion required — panics if wrong type

// Post-generics (Go 1.18+): type-safe, no boxing overhead
type TypedStack[T any] struct {
    items []T
}

func (s *TypedStack[T]) Push(v T) { s.items = append(s.items, v) }
func (s *TypedStack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 { return zero, false }
    n := len(s.items)
    v := s.items[n-1]
    s.items = s.items[:n-1]
    return v, true
}

// Now type-safe at compile time:
s2 := &TypedStack[int]{}
s2.Push(42)
// s2.Push("hello") // COMPILE ERROR — type mismatch
v2, _ := s2.Pop() // returns int directly, no assertion needed
```

Performance implication: storing a value in `interface{}` may cause **boxing** (heap allocation) for non-pointer types. Generics avoid this by generating specialized code.

---

## Trap 6: Method set of a pointer vs non-pointer type in interface satisfaction

```go
type Resetter interface {
    Reset()
}

type Buffer struct {
    data []byte
}

func (b *Buffer) Reset() { b.data = b.data[:0] }

// Buffer does NOT satisfy Resetter (only *Buffer does)
var _ Resetter = &Buffer{} // OK
// var _ Resetter = Buffer{}  // COMPILE ERROR

// This matters in function signatures:
func resetAll(items []Resetter) {
    for _, item := range items {
        item.Reset()
    }
}

buffers := []*Buffer{{}, {}}
resetters := make([]Resetter, len(buffers))
for i, b := range buffers {
    resetters[i] = b // &Buffer satisfies Resetter
}
resetAll(resetters)

// But:
vals := []Buffer{{}, {}}
// Cannot create []Resetter from []Buffer — Buffer doesn't satisfy Resetter
```

---

## Trap 7: Struct tags — what if the tag is malformed?

```go
type User struct {
    Name string `json: "name"` // WRONG: space after "json:"
    Age  int    `json:"age"`   // CORRECT
}

// The malformed tag doesn't cause a compile error!
// The json package silently ignores the malformed tag
// 'go vet' will catch this — run it in CI!

u := User{Name: "Alice", Age: 30}
b, _ := json.Marshal(u)
// {"Name":"Alice","age":30}
// Name is NOT renamed because the tag is ignored
// Age IS renamed because the tag is correct

// Always use go vet or a linter to validate struct tags
// go vet ./...
```

---

## Trap 8: Can you add methods to types from other packages?

No, and the error message is "cannot define new methods on non-local type".

```go
// You cannot do this:
func (t time.Time) HumanReadable() string {  // COMPILE ERROR
    return t.Format("January 2, 2006")
}

// You CAN define a new named type based on time.Time:
type HumanTime time.Time

func (h HumanTime) String() string {
    return time.Time(h).Format("January 2, 2006")
}

// Or use a wrapper function instead:
func humanReadable(t time.Time) string {
    return t.Format("January 2, 2006")
}
```

---

## Trap 9: sync.Once and panic — the function is never called again

```go
var once sync.Once
var db *Database

func initDB() {
    once.Do(func() {
        // If this panics, once.Do will never run the function again
        // but the Once is permanently "done"
        db = connectToDB() // suppose this panics
    })
}

// After a panic in once.Do, the function is permanently marked as done
// Subsequent calls to once.Do are silently no-ops
// db remains nil forever

// Safe pattern: recover inside once.Do
func initDBSafe() (err error) {
    once.Do(func() {
        defer func() {
            if r := recover(); r != nil {
                err = fmt.Errorf("DB init panicked: %v", r)
            }
        }()
        db = connectToDB()
    })
    return err
}

// Or use a dedicated error variable:
var (
    once   sync.Once
    db     *Database
    dbErr  error
)

func GetDB() (*Database, error) {
    once.Do(func() {
        db, dbErr = connectToDB()
    })
    return db, dbErr
}
```

---

## Trap 10: Circular embedding — not allowed

```go
// This would create an infinite type:
type A struct {
    B // COMPILE ERROR if B contains A
}

type B struct {
    A // circular embedding — impossible
}

// Pointers break the cycle (and are how recursive types work):
type TreeNode struct {
    Value    int
    Children []*TreeNode // pointer — allowed
}

// Similarly:
type A struct {
    B *B // pointer to B — allowed
}
type B struct {
    A *A // pointer to A — allowed
}
```

---

## Trap 11: Interface with multiple types in type assertion — all or nothing

```go
type ReadWriter interface {
    io.Reader
    io.Writer
}

var rw ReadWriter = os.Stdin

// You can assert to component interfaces:
r, ok := rw.(io.Reader)  // ok = true — ReadWriter embeds Reader
w, ok := rw.(io.Writer)  // ok = true — ReadWriter embeds Writer

// You can assert to the concrete type:
f, ok := rw.(*os.File)   // ok = true (os.Stdin is *os.File)

// But assigning back only works if the concrete type still satisfies the interface:
var f *os.File = os.Stdin
var _ ReadWriter = f  // OK — *os.File implements both Read and Write
```

---

## Trap 12: Promoted methods and interface satisfaction — the subtle rule

```go
type Inner struct{}
func (Inner) Read(p []byte) (int, error) { return 0, io.EOF }

type Outer struct {
    Inner // Read is promoted to Outer
}

// Does Outer satisfy io.Reader?
var _ io.Reader = Outer{}  // YES — promoted method counts!
var _ io.Reader = &Outer{} // YES — pointer type also gets it

// But: if Outer defines its OWN Read, it shadows Inner.Read
func (o Outer) Read(p []byte) (int, error) {
    // use inner.Read as well:
    return o.Inner.Read(p)
}

// The method set used for interface satisfaction is the outermost layer's
// method set — promoted methods are included unless shadowed
```
