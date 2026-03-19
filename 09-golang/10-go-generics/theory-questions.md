# Go Generics — Theory Questions

---

## Generics Fundamentals

**Q1. What are generics in Go and when were they introduced?**

Generics (formally: **type parameters**) were introduced in **Go 1.18** (released March 2022). They allow writing functions and types that work with any type satisfying a given constraint, without code duplication or losing type safety.

Before generics:
```go
// You needed separate functions or used interface{}:
func MaxInt(a, b int) int {
    if a > b { return a }
    return b
}
func MaxFloat64(a, b float64) float64 {
    if a > b { return a }
    return b
}
// Or: lose type safety:
func Max(a, b interface{}) interface{} { /* ... */ }
```

With generics:
```go
func Max[T constraints.Ordered](a, b T) T {
    if a > b { return a }
    return b
}

Max(3, 4)       // T inferred as int
Max(3.14, 2.71) // T inferred as float64
Max("foo", "bar") // T inferred as string
```

---

**Q2. What is a type parameter and how is it declared?**

A type parameter is a placeholder for a concrete type, declared in square brackets `[T Constraint]` after the function or type name.

```go
// Function with one type parameter
func Print[T any](v T) {
    fmt.Println(v)
}

// Function with multiple type parameters
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Generic type (struct)
type Pair[T, U any] struct {
    First  T
    Second U
}

p := Pair[string, int]{First: "age", Second: 30}
```

---

**Q3. What is `any` as a constraint?**

`any` is an alias for `interface{}` — it means the type parameter can be ANY type. It was added in Go 1.18 alongside generics.

```go
// any = no restriction on T
func Contains[T any](slice []T, item T) bool {
    // But wait — you can't compare T values with == unless T is comparable!
    // This won't compile:
    for _, v := range slice {
        if v == item { // ERROR: cannot compare T with ==
            return true
        }
    }
    return false
}

// Fix: use comparable constraint
func Contains[T comparable](slice []T, item T) bool {
    for _, v := range slice {
        if v == item { // OK: T is guaranteed comparable
            return true
        }
    }
    return false
}
```

---

**Q4. What is the `comparable` constraint?**

`comparable` is a built-in constraint satisfied by all types that support `==` and `!=` operators. This includes: integers, floats, strings, booleans, arrays (of comparable elements), pointers, channels, and structs with only comparable fields.

NOT comparable: slices, maps, functions.

```go
// Generic map/set operations require comparable keys:
func SetContains[K comparable](set map[K]struct{}, key K) bool {
    _, ok := set[key]
    return ok
}

// Generic cache:
type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]V
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}
```

---

**Q5. What is a union type constraint and how do you define one?**

Union constraints use `|` to list specific types that satisfy the constraint:

```go
// Only int, float64, or string are allowed
type Numeric interface {
    int | int32 | int64 | float32 | float64
}

func Sum[T Numeric](values []T) T {
    var total T
    for _, v := range values {
        total += v
    }
    return total
}

Sum([]int{1, 2, 3})       // T = int
Sum([]float64{1.1, 2.2})  // T = float64

// Cannot use Sum with a custom type named "MyInt" unless you use ~
```

---

**Q6. What is the `~T` (tilde) operator in constraints?**

`~T` means: any type whose **underlying type** is `T`. This includes `T` itself AND any named types built on top of `T`.

```go
// Without ~: only exact int is allowed
type StrictInts interface { int }

type MyInt int
Sum[StrictInts]([]MyInt{1, 2, 3}) // COMPILE ERROR: MyInt doesn't satisfy StrictInts

// With ~: int AND any type with underlying type int
type Integers interface { ~int | ~int32 | ~int64 }

type UserID int    // underlying type is int
type ProductID int // underlying type is int

func AddIDs[T Integers](a, b T) T { return a + b }
AddIDs(UserID(1), UserID(2))    // OK: UserID's underlying type is int
AddIDs(ProductID(3), ProductID(4)) // OK: same reason
```

**The `~` is critical for making constraints work with user-defined types.** Most real-world constraints use `~`:

```go
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
    ~float32 | ~float64 |
    ~string
}
```

---

**Q7. What is the constraints package (golang.org/x/exp/constraints)?**

The `golang.org/x/exp/constraints` package (experimental, will move to stdlib) provides common constraints:

```go
import "golang.org/x/exp/constraints"

// constraints.Ordered: all types supporting < > <= >=
type Ordered interface {
    Integer | Float | ~string
}

// constraints.Integer: all integer types (signed and unsigned)
type Integer interface {
    Signed | Unsigned
}

// constraints.Float: float32 and float64
type Float interface {
    ~float32 | ~float64
}

// constraints.Complex: complex64 and complex128
type Complex interface {
    ~complex64 | ~complex128
}

// Usage:
func Min[T constraints.Ordered](a, b T) T {
    if a < b { return a }
    return b
}
```

In Go 1.21, the `cmp` package was added to stdlib with `cmp.Ordered` (equivalent to `constraints.Ordered`).

---

**Q8. How does type inference work in Go generics?**

The compiler infers type parameters from the argument types, so you often don't need to specify them explicitly:

```go
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Explicit:
strs := Map[int, string]([]int{1, 2, 3}, strconv.Itoa)

// Inferred: compiler sees []int → T=int; strconv.Itoa returns string → U=string
strs := Map([]int{1, 2, 3}, strconv.Itoa)

// Type inference works when:
// - Type parameters can be deduced from function arguments
// - All type parameters can be resolved unambiguously

// Type inference does NOT work for:
// - Generic types instantiated without arguments
type Stack[T any] struct { items []T }
s := Stack[int]{}  // MUST specify — compiler can't infer from constructor call with no args
```

---

**Q9. How do you implement a generic Stack?**

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    last := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return last, true
}

func (s *Stack[T]) Peek() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) Len() int { return len(s.items) }

// Usage:
intStack := &Stack[int]{}
intStack.Push(1)
intStack.Push(2)
v, ok := intStack.Pop() // v=2, ok=true

strStack := &Stack[string]{}
strStack.Push("hello")
strStack.Push("world")
```

---

**Q10. How do you implement a generic Set?**

```go
type Set[T comparable] struct {
    m map[T]struct{}
}

func NewSet[T comparable](items ...T) *Set[T] {
    s := &Set[T]{m: make(map[T]struct{}, len(items))}
    for _, item := range items {
        s.Add(item)
    }
    return s
}

func (s *Set[T]) Add(item T) {
    s.m[item] = struct{}{}
}

func (s *Set[T]) Remove(item T) {
    delete(s.m, item)
}

func (s *Set[T]) Contains(item T) bool {
    _, ok := s.m[item]
    return ok
}

func (s *Set[T]) Len() int { return len(s.m) }

func (s *Set[T]) ToSlice() []T {
    result := make([]T, 0, len(s.m))
    for k := range s.m {
        result = append(result, k)
    }
    return result
}

func (s *Set[T]) Union(other *Set[T]) *Set[T] {
    result := NewSet[T]()
    for k := range s.m { result.Add(k) }
    for k := range other.m { result.Add(k) }
    return result
}

func (s *Set[T]) Intersection(other *Set[T]) *Set[T] {
    result := NewSet[T]()
    for k := range s.m {
        if other.Contains(k) {
            result.Add(k)
        }
    }
    return result
}
```

---

**Q11. What are the slices and maps packages added in Go 1.21?**

Go 1.21 added `slices` and `maps` packages using generics to eliminate common boilerplate:

```go
import (
    "slices"
    "maps"
    "cmp"
)

// slices package:
nums := []int{3, 1, 4, 1, 5, 9, 2, 6}

slices.Sort(nums)                    // sort in place
slices.SortFunc(nums, cmp.Compare)   // sort with custom comparator
slices.Contains(nums, 5)             // true
slices.Index(nums, 4)                // index of 4 (or -1)
slices.Reverse(nums)                 // reverse in place
slices.Compact(nums)                 // remove consecutive duplicates
slices.Equal(nums, []int{1,2,3})     // element-wise equality
slices.Max(nums)                     // max element (O(n))
slices.Min(nums)                     // min element
slices.ContainsFunc(nums, func(n int) bool { return n > 5 }) // any > 5?

// maps package:
m := map[string]int{"a": 1, "b": 2, "c": 3}

maps.Keys(m)          // []string{"a", "b", "c"} (unordered)
maps.Values(m)        // []int{1, 2, 3} (unordered)
maps.Clone(m)         // shallow copy
maps.Copy(dst, src)   // copy src entries into dst
maps.Equal(m, m2)     // key-value equality
maps.DeleteFunc(m, func(k string, v int) bool { return v < 2 }) // delete if < 2
```

---

**Q12. When should you use generics vs interface{}?**

| Situation | Use | Reason |
|-----------|-----|--------|
| Type must be determined at compile time | Generics | Type safety, no boxing |
| Container (Stack, Set, Queue) | Generics | Type-safe elements, no casting |
| Utility functions (Map, Filter, Reduce) | Generics | Work with any type, type-safe |
| Plugin architecture (unknown types at compile time) | interface{} or interfaces | Extensibility |
| Behavior-based abstraction (`io.Reader`, `http.Handler`) | Interface | Behavior, not data type |
| JSON/gob unmarshaling into unknown types | interface{} | Runtime type determination |
| Recursive data structures with mixed types | interface{} | Simpler than complex constraints |

```go
// USE GENERICS: type-safe data structure
func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

// USE INTERFACE: behavior abstraction
type Renderer interface {
    Render(w io.Writer) error
}
// You don't know all the types upfront — any struct can implement this
```

---

**Q13. What are the limitations of Go generics?**

1. **No specialization**: Cannot write different code for specific type parameter values
2. **No operator overloading**: Cannot define `+` for custom types
3. **No methods with additional type parameters**: Only the receiver type can introduce type parameters

```go
// LIMITATION 1: No specialization
func Process[T any](v T) {
    // Can't write: if T is int { do int-specific thing }
    // Workaround: use type switch on interface{}
}

// LIMITATION 2: Methods cannot have their own type parameters
type MyList[T any] struct { items []T }

// ILLEGAL: methods cannot introduce new type parameters
func (l *MyList[T]) Transform[U any](fn func(T) U) []U { ... } // compile error

// Workaround: make it a standalone function
func Transform[T, U any](list *MyList[T], fn func(T) U) []U { ... }

// LIMITATION 3: Cannot use type assertions on type parameters
func doSomething[T any](v T) {
    if s, ok := v.(string); ok { // compile error: cannot type assert on T
        _ = s
    }
    // Workaround: convert to interface{} first
    if s, ok := any(v).(string); ok {
        _ = s
    }
}
```

---

**Q14. How do Go generics compare to Java generics (stenciling vs type erasure)?**

**Java: Type Erasure**
- Type parameters are erased at compile time to `Object`
- All `List<String>` and `List<Integer>` use the same bytecode
- Runtime reflection can't distinguish `List<String>` from `List<Integer>`
- Boxing: `int` becomes `Integer` (heap allocation)

**Go: GCShape Stenciling**
- The compiler generates shared code for types with the same **GC shape** (memory layout + pointer structure)
- All pointer types share one instantiation (they have the same GC shape)
- Each distinct value type gets its own instantiation
- No boxing: `int` stays an `int` in memory

```go
// Go: separate instantiation for int (value type)
Stack[int]    // dedicated code path for int
Stack[float64] // dedicated code path for float64

// Go: shared instantiation for all pointers (same GC shape)
Stack[*User]  // same code as Stack[*Product] (both are just pointer-sized words)
Stack[*Product]

// Java equivalent would box both int and float64 to Object:
Stack<Integer> // Integer is heap-allocated wrapper for int
Stack<Float>   // same boxing issue
```

**Practical implications:**
- Go generics are faster than Java for primitive types (no boxing overhead)
- Go generics compile to fewer unique instantiations than full C++ monomorphization
- Go's approach is a pragmatic middle ground

---

**Q15. What is a generic Queue implementation?**

```go
type Queue[T any] struct {
    items []T
}

func (q *Queue[T]) Enqueue(item T) {
    q.items = append(q.items, item)
}

func (q *Queue[T]) Dequeue() (T, bool) {
    if len(q.items) == 0 {
        var zero T
        return zero, false
    }
    item := q.items[0]
    q.items = q.items[1:] // naive: causes memory leak via sub-slice
    return item, true
}

// Better: ring buffer or two-slice approach
type RingQueue[T any] struct {
    buf  []T
    head int
    tail int
    size int
}

func NewRingQueue[T any](capacity int) *RingQueue[T] {
    return &RingQueue[T]{buf: make([]T, capacity)}
}

func (q *RingQueue[T]) Enqueue(item T) bool {
    if q.size == len(q.buf) {
        return false // full
    }
    q.buf[q.tail] = item
    q.tail = (q.tail + 1) % len(q.buf)
    q.size++
    return true
}

func (q *RingQueue[T]) Dequeue() (T, bool) {
    if q.size == 0 {
        var zero T
        return zero, false
    }
    item := q.buf[q.head]
    q.head = (q.head + 1) % len(q.buf)
    q.size--
    return item, true
}
```

---

**Q16. How do you implement generic Map, Filter, and Reduce functions?**

```go
// Map: transform every element
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Filter: keep elements satisfying a predicate
func Filter[T any](slice []T, fn func(T) bool) []T {
    var result []T
    for _, v := range slice {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

// Reduce: fold slice into a single value
func Reduce[T, Acc any](slice []T, initial Acc, fn func(Acc, T) Acc) Acc {
    acc := initial
    for _, v := range slice {
        acc = fn(acc, v)
    }
    return acc
}

// Usage:
nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

// Square each number
squares := Map(nums, func(n int) int { return n * n })

// Keep only even numbers
evens := Filter(nums, func(n int) bool { return n%2 == 0 })

// Sum all
total := Reduce(nums, 0, func(acc, n int) int { return acc + n })

// Chain: sum of squares of even numbers
result := Reduce(
    Map(Filter(nums, func(n int) bool { return n%2 == 0 }),
        func(n int) int { return n * n }),
    0,
    func(acc, n int) int { return acc + n },
)
// = 4 + 16 + 36 + 64 + 100 = 220
```

---

**Q17. What is the performance impact of generics vs interface{}?**

```go
// Benchmark setup:
type Numeric interface { ~int | ~float64 }

func SumGeneric[T Numeric](s []T) T {
    var total T
    for _, v := range s { total += v }
    return total
}

func SumInterface(s []interface{}) float64 {
    var total float64
    for _, v := range s { total += v.(float64) } // type assertion = overhead
    return total
}

// Results (approximate, n=10000):
// SumGeneric[int]:    ~2 ns/op, 0 allocs    (inlineable, no boxing)
// SumInterface:      ~25 ns/op, 1 alloc/op  (type assertion + boxing)
```

For types with the same GC shape (e.g., all pointers), generics may compile to shared code — similar performance to interface. For value types (int, float, structs), generics get their own instantiation and are typically faster.

---

**Q18. What is a generic Result type?**

Inspired by Rust's `Result<T, E>`, this eliminates the repetitive `if err != nil` pattern:

```go
type Result[T any] struct {
    value T
    err   error
}

func Ok[T any](v T) Result[T]    { return Result[T]{value: v} }
func Err[T any](e error) Result[T] { return Result[T]{err: e} }

func (r Result[T]) IsOk() bool    { return r.err == nil }
func (r Result[T]) IsErr() bool   { return r.err != nil }

func (r Result[T]) Unwrap() T {
    if r.err != nil {
        panic("unwrap on error result: " + r.err.Error())
    }
    return r.value
}

func (r Result[T]) UnwrapOr(def T) T {
    if r.err != nil {
        return def
    }
    return r.value
}

func (r Result[T]) Error() error { return r.err }

// Map applies fn to the value if Ok
func MapResult[T, U any](r Result[T], fn func(T) Result[U]) Result[U] {
    if r.IsErr() {
        return Err[U](r.err)
    }
    return fn(r.value)
}

// Usage:
func divide(a, b float64) Result[float64] {
    if b == 0 {
        return Err[float64](errors.New("division by zero"))
    }
    return Ok(a / b)
}

r := divide(10, 2)
fmt.Println(r.Unwrap()) // 5

r2 := divide(10, 0)
fmt.Println(r2.UnwrapOr(-1)) // -1 (no panic)
```

---

**Q19. What is a generic Option type?**

```go
type Option[T any] struct {
    value    T
    hasValue bool
}

func Some[T any](v T) Option[T]  { return Option[T]{value: v, hasValue: true} }
func None[T any]() Option[T]     { return Option[T]{} }

func (o Option[T]) IsSome() bool { return o.hasValue }
func (o Option[T]) IsNone() bool { return !o.hasValue }

func (o Option[T]) Unwrap() T {
    if !o.hasValue {
        panic("unwrap on None")
    }
    return o.value
}

func (o Option[T]) UnwrapOr(def T) T {
    if !o.hasValue { return def }
    return o.value
}

// Usage:
func findUser(id int) Option[User] {
    user, ok := db[id]
    if !ok {
        return None[User]()
    }
    return Some(user)
}

opt := findUser(42)
name := opt.Map(func(u User) string { return u.Name }).UnwrapOr("unknown")
```

---

**Q20. How do you write a generic binary search?**

```go
import "cmp"

// BinarySearch finds the index of target in a sorted slice.
// Returns (index, true) if found, (insertion-point, false) if not found.
func BinarySearch[T cmp.Ordered](sorted []T, target T) (int, bool) {
    lo, hi := 0, len(sorted)
    for lo < hi {
        mid := lo + (hi-lo)/2
        c := cmp.Compare(sorted[mid], target)
        switch {
        case c < 0:
            lo = mid + 1
        case c > 0:
            hi = mid
        default:
            return mid, true // found
        }
    }
    return lo, false // not found; lo is insertion point
}

// Usage:
sorted := []int{1, 3, 5, 7, 9, 11}
idx, found := BinarySearch(sorted, 7) // idx=3, found=true
idx, found = BinarySearch(sorted, 6)  // idx=3, found=false (would insert at 3)

// Works with any ordered type:
words := []string{"apple", "banana", "cherry", "date"}
idx, found = BinarySearch(words, "banana") // idx=1, found=true
```

Note: Go 1.21's `slices` package has `slices.BinarySearch` built in.
