# Go Generics — Structured Answers

---

## Answer 1: What Are Generics in Go and When Were They Introduced?

**One-sentence answer:** Generics (type parameters) were introduced in Go 1.18 (March 2022), allowing functions and types to be written once and work safely with any type that satisfies a specified constraint.

**The problem they solve:**

```go
// Before generics: code duplication for each type
func MinInt(a, b int) int         { if a < b { return a }; return b }
func MinFloat64(a, b float64) float64 { if a < b { return a }; return b }
func MinString(a, b string) string    { if a < b { return a }; return b }

// OR: type safety lost with interface{}
func Min(a, b interface{}) interface{} {
    // complex type assertion logic...
    // runtime panics possible
    // no compile-time error checking
}

// With generics: one implementation, compile-time type safety
func Min[T constraints.Ordered](a, b T) T {
    if a < b { return a }
    return b
}

Min(3, 4)           // T = int
Min(3.14, 2.71)     // T = float64
Min("foo", "bar")   // T = string
Min(true, false)    // COMPILE ERROR: bool does not implement Ordered ← caught early
```

**Key design goals of Go generics:**
- Compile-time type safety (no casting at runtime)
- Zero boxing overhead for value types
- Works with existing interfaces (constraints are interfaces)
- Backward compatible (old code unchanged)

---

## Answer 2: What Is a Type Constraint and How Do You Define One?

**One-sentence answer:** A type constraint is an interface that restricts what types can be used as a type parameter — it can specify methods the type must implement, specific underlying types, or combinations of both.

**Three forms of constraints:**

```go
// Form 1: Method-based (just like regular interfaces)
type Stringer interface {
    String() string
}

func Print[T Stringer](v T) {
    fmt.Println(v.String())
}

// Form 2: Union type sets (type literals, generics-specific)
type Number interface {
    ~int | ~int32 | ~int64 | ~float32 | ~float64
}

func Sum[T Number](items []T) T {
    var total T
    for _, v := range items { total += v }
    return total
}

// Form 3: Combined (methods + type set)
type StringLike interface {
    ~string
    String() string  // must also have a String() method
}

// Built-in constraints:
// any         = interface{} — no restrictions
// comparable  = supports == and != (built-in, not an interface you can define)
```

**Inline vs named constraints:**

```go
// Named constraint (reusable):
type Numeric interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64
}

func Abs[T Numeric](v T) T {
    if v < 0 { return -v }
    return v
}

// Inline constraint (one-off use):
func Add[T interface{ ~int | ~float64 }](a, b T) T {
    return a + b
}
```

---

## Answer 3: Implement a Generic, Reusable Result[T] Type

**One-sentence answer:** `Result[T]` is a generic type that encapsulates either a success value of type T or an error, eliminating the possibility of accidentally using a value from a failed operation.

```go
package result

// Result holds either a value of type T or an error.
type Result[T any] struct {
    value T
    err   error
}

// Constructors
func Ok[T any](v T) Result[T]      { return Result[T]{value: v} }
func Err[T any](e error) Result[T] { return Result[T]{err: e} }

// Predicates
func (r Result[T]) IsOk() bool  { return r.err == nil }
func (r Result[T]) IsErr() bool { return r.err != nil }

// Value extraction
func (r Result[T]) Unwrap() T {
    if r.err != nil {
        panic("result.Unwrap() called on Err: " + r.err.Error())
    }
    return r.value
}

func (r Result[T]) UnwrapOr(def T) T {
    if r.err != nil { return def }
    return r.value
}

func (r Result[T]) UnwrapOrElse(fn func(error) T) T {
    if r.err != nil { return fn(r.err) }
    return r.value
}

func (r Result[T]) Error() error { return r.err }

// Map transforms the value if Ok
func Map[T, U any](r Result[T], fn func(T) Result[U]) Result[U] {
    if r.IsErr() {
        return Err[U](r.err)
    }
    return fn(r.value)
}

// AndThen chains fallible operations (monad bind)
func (r Result[T]) AndThen(fn func(T) error) Result[T] {
    if r.IsErr() { return r }
    if err := fn(r.value); err != nil {
        return Err[T](err)
    }
    return r
}

// ToTuple converts to (T, error) for use with standard Go patterns
func (r Result[T]) ToTuple() (T, error) { return r.value, r.err }

// Usage examples:
func divide(a, b float64) Result[float64] {
    if b == 0 {
        return Err[float64](errors.New("division by zero"))
    }
    return Ok(a / b)
}

func parseInt(s string) Result[int] {
    n, err := strconv.Atoi(s)
    if err != nil {
        return Err[int](fmt.Errorf("invalid number %q: %w", s, err))
    }
    return Ok(n)
}

// Chaining results:
func parseAndDivide(input string, divisor float64) Result[float64] {
    return Map(parseInt(input), func(n int) Result[float64] {
        return divide(float64(n), divisor)
    })
}

r := parseAndDivide("42", 7.0)
fmt.Println(r.Unwrap())      // 6.0

r2 := parseAndDivide("abc", 7.0)
fmt.Println(r2.IsErr())      // true
fmt.Println(r2.Error())      // invalid number "abc": strconv.Atoi: ...
fmt.Println(r2.UnwrapOr(0)) // 0
```

---

## Answer 4: What Is the ~ (Tilde) Operator in Constraints?

**One-sentence answer:** `~T` in a constraint means "any type whose underlying type is T" — this includes T itself plus any named types derived from T, making constraints work with domain-specific named types.

**Underlying type explained:**

```go
type Celsius    float64  // underlying type: float64
type Fahrenheit float64  // underlying type: float64
type UserID     int      // underlying type: int
type Dollars    float64  // underlying type: float64
```

**Without ~: narrow (excludes derived types):**

```go
type NumberConstraint interface { int | float64 }

func Double[T NumberConstraint](v T) T { return v * 2 }

Double(42)        // OK: untyped int → int
Double(3.14)      // OK: untyped float → float64
Double(UserID(5)) // COMPILE ERROR: UserID does not satisfy NumberConstraint
```

**With ~: broad (includes derived types):**

```go
type NumberConstraint interface { ~int | ~float64 }

func Double[T NumberConstraint](v T) T { return v * 2 }

Double(UserID(5))    // OK: UserID has underlying type int
Double(Celsius(37))  // OK: Celsius has underlying type float64
Double(Dollars(9.99)) // OK: Dollars has underlying type float64
```

**Why this matters for domain modeling:**

```go
// Type-safe arithmetic with domain types:
type Money struct {
    amount   Dollars
    currency string
}

func Scale[T interface{ ~float64 }](v T, factor float64) T {
    return T(float64(v) * factor)
}

salary := Dollars(50000)
bonus := Scale(salary, 0.1) // Dollars(5000) — type preserved!
```

Without `~`, you'd have to convert to `float64`, do the math, and convert back — losing type safety. With `~`, the constraint covers all float64-based domain types.

---

## Answer 5: How Do Go Generics Compare to Java Generics (Type Erasure vs Stenciling)?

**One-sentence answer:** Java erases type parameters at runtime (all become `Object`), requiring boxing for primitives and losing type info; Go uses GCShape stenciling — generating code per unique memory layout, preserving performance for value types with no boxing.

**Java generics (type erasure):**

```java
// Java: List<Integer> and List<String> compile to the same bytecode
List<Integer> nums = new ArrayList<>();
nums.add(42);  // 42 (int) is boxed to Integer (heap object) — allocation!

// At runtime, it's just List — no Integer info
// Can't write: if (list instanceof List<Integer>) — erased!
// int → Integer boxing causes GC pressure in hot paths
```

**Go generics (GCShape stenciling):**

```go
// Go: shares instantiation per GC shape (memory layout)

// Value types: separate instantiation
type Stack[T any] struct { items []T }
var intStack Stack[int]    // 8-byte values
var f32Stack Stack[float32] // 4-byte values
// ^ These get DIFFERENT machine code (different GC shapes)

// Pointer types: shared instantiation
var uStack Stack[*User]    // 8-byte pointer
var pStack Stack[*Product] // 8-byte pointer
// ^ These share the SAME machine code (same GC shape: 8-byte pointer)

// No boxing: int is still an int on the stack
var s Stack[int]
s.Push(42) // 42 stays as int, no heap allocation
```

**Performance comparison:**

| Operation | Java `List<Integer>` | Go `Stack[int]` |
|-----------|---------------------|-----------------|
| Push integer | `new Integer(42)` — heap alloc | stack alloc — zero overhead |
| Pop integer | unbox Integer → int | direct int return |
| Memory per element | ~16 bytes (Integer object) | 8 bytes |

**What Go can't do that C++ templates can:**
- Full monomorphization per type (Go shares for same GC shape)
- Specialization (different code for specific T values)

---

## Answer 6: What Are the slices and maps Packages (Go 1.21)?

**One-sentence answer:** Go 1.21 added the `slices` and `maps` standard library packages using generics, providing type-safe utility functions for common slice and map operations that previously required manual implementation or unsafe casts.

```go
import (
    "slices"
    "maps"
    "cmp"
)

// ── slices package ───────────────────────────────────────────────────

nums := []int{3, 1, 4, 1, 5, 9, 2, 6, 5, 3}

// Sorting
slices.Sort(nums)                                    // [1 1 2 3 3 4 5 5 6 9]
slices.SortFunc(nums, func(a, b int) int {           // custom comparator
    return cmp.Compare(b, a)                         // descending
})
slices.IsSorted(nums)                                // bool
slices.SortStableFunc(nums, cmp.Compare)             // stable sort

// Searching
slices.Contains(nums, 5)                             // true
slices.Index(nums, 4)                                // index or -1
slices.IndexFunc(nums, func(n int) bool { return n > 5 }) // first > 5
slices.ContainsFunc(nums, func(n int) bool { return n < 0 }) // false (no negatives)

// Slices operations
slices.Compact(nums)                                 // remove consecutive dupes
slices.CompactFunc(nums, func(a, b int) bool { return a == b })
slices.Clip(nums)                                    // trim excess capacity
slices.Reverse(nums)                                 // in-place reverse
slices.Equal(nums, []int{1, 2, 3})                   // element-wise ==
slices.EqualFunc(nums, other, func(a, b int) bool { return a == b })

// Functional
slices.Max([]int{3, 1, 4})                           // 4
slices.Min([]int{3, 1, 4})                           // 1
slices.MaxFunc(data, cmp.Compare)                    // with custom comparator

// Binary search (on sorted slice)
idx, found := slices.BinarySearch([]int{1, 2, 4, 8}, 4) // 2, true
slices.BinarySearchFunc(data, target, cmp.Compare)

// ── maps package ─────────────────────────────────────────────────────

m := map[string]int{"a": 1, "b": 2, "c": 3}

maps.Keys(m)            // []string (unordered)
maps.Values(m)          // []int (unordered)
maps.Clone(m)           // shallow copy
maps.Copy(dst, src)     // merge src into dst

// Delete entries matching predicate
maps.DeleteFunc(m, func(k string, v int) bool {
    return v < 2        // delete if value < 2
})

maps.Equal(m, map[string]int{"a": 1, "b": 2}) // key-value equality
maps.EqualFunc(m, other, func(v1, v2 int) bool { return v1 == v2 })

// Collect from iterator (Go 1.23+)
// maps.Collect(iter.Seq2[K, V]) map[K]V
```

---

## Answer 7: When Should You Use Generics vs interface{}?

**One-sentence answer:** Use generics when the type must be known at compile time for type safety or performance; use interfaces when you need runtime polymorphism, extensibility, or behavior-based abstraction.

**Decision matrix:**

```go
// USE GENERICS when:

// 1. Building type-safe data structures
type Queue[T any] struct { items []T }
// Without: Queue[interface{}] requires cast on every Dequeue

// 2. Writing utility functions over slices/maps
func Keys[K comparable, V any](m map[K]V) []K { /* ... */ }
// Without: func Keys(m map[string]int) []string — one version per type

// 3. Functional primitives (Map, Filter, Reduce)
func Map[T, U any](slice []T, fn func(T) U) []U { /* ... */ }

// 4. Avoiding boxing for performance in hot paths
func Sum[T Numeric](items []T) T { /* ... */ }
// Sum[int] stays as int operations — no interface boxing


// USE INTERFACES when:

// 1. Behavior abstraction (multiple types share behavior)
type Writer interface { Write(p []byte) (n int, err error) }
func LogTo(w Writer, msg string) { w.Write([]byte(msg + "\n")) }
// Works with os.File, bytes.Buffer, net.Conn, etc.

// 2. Plugin architecture (types not known at compile time)
type Plugin interface { Execute(ctx context.Context) error }
var plugins []Plugin  // registered at runtime

// 3. Dependency injection / mocking in tests
type UserRepo interface { Get(ctx context.Context, id string) (*User, error) }
// Implementation can be postgres, in-memory, or mock

// 4. Heterogeneous collections
items := []interface{}{"hello", 42, true}  // mixed types

// AVOID:
// Replacing well-designed interfaces with generics for no benefit
type Processor[T any] interface { Process(T) }
// vs
type Processor interface { Process(v interface{}) }
// If there's no type-safety benefit, the generic version adds complexity

// Converting interface{} code to generics mechanically
// Generics add value when you have REPEATED code over multiple types
// If you only have one use case, generics add cognitive overhead
```

---

## Answer 8: What Are the Current Limitations of Go Generics?

**One-sentence answer:** Go generics cannot specialize behavior per type, methods cannot introduce new type parameters, type switches on type parameters require going through `any`, and there is no operator overloading.

**Limitation 1: No method type parameters:**
```go
type Container[T any] struct{ items []T }

// ILLEGAL: transform to a different type
func (c Container[T]) MapTo[U any](fn func(T) U) Container[U] { ... }

// WORKAROUND: standalone function
func MapContainer[T, U any](c Container[T], fn func(T) U) Container[U] { ... }
```

**Limitation 2: No specialization:**
```go
// Cannot do different things based on T's type:
func Process[T any](v T) {
    // "if T is int, do this; if T is string, do that" — not directly possible
    // Must go through any(v).(type) switch
}
```

**Limitation 3: No operator overloading:**
```go
type Vector2D[T Numeric] struct{ X, Y T }

// Cannot define: v1 + v2 using + operator
// Must use a method:
func (v Vector2D[T]) Add(other Vector2D[T]) Vector2D[T] {
    return Vector2D[T]{v.X + other.X, v.Y + other.Y}
}
```

**Limitation 4: comparable is not fully introspectable:**
```go
// You can use == but not reflect.DeepEqual via constraint
// comparable includes struct, array — but not slice, map
```

**Limitation 5: Generic types cannot be used as type assertions:**
```go
var v any = Stack[int]{}
_, ok := v.(Stack[int])  // OK — specific instantiation
_, ok = v.(Stack[any])   // COMPILE ERROR — Stack[any] is not an interface
```

**Limitation 6: No covariance/contravariance:**
```go
// In Java: List<Animal> cannot receive List<Dog> even if Dog extends Animal
// In Go: same — Stack[Dog] is not assignable to Stack[Animal]
// (This is actually type-safe, but can require more explicit conversions)
```
