# Go Generics — Follow-Up Traps

---

## Trap 1: Cannot Use Type Parameter as Map Key Without comparable

**The trap:** "I made a generic function with `[T any]` and used T as a map key. It doesn't compile."

**The problem:** Not all types are comparable. Slices, maps, and functions cannot be used as map keys. If `T` is `any`, the compiler can't guarantee `T` is comparable.

```go
// COMPILE ERROR: cannot use T as map key (T is any)
func CountOccurrences[T any](items []T) map[T]int {
    counts := make(map[T]int)  // ERROR: invalid map key type T
    for _, item := range items {
        counts[item]++          // ERROR: cannot use T as key
    }
    return counts
}

// FIX: require comparable
func CountOccurrences[T comparable](items []T) map[T]int {
    counts := make(map[T]int)  // OK: T is guaranteed comparable
    for _, item := range items {
        counts[item]++
    }
    return counts
}

// Works with:
CountOccurrences([]int{1, 2, 1, 3, 2, 1})     // int is comparable
CountOccurrences([]string{"a", "b", "a"})       // string is comparable
CountOccurrences([]struct{ X, Y int }{{1, 2}})  // struct with comparable fields

// Still fails at call site with non-comparable types:
CountOccurrences([][]int{{1, 2}, {1, 2}})  // compile error: []int does not implement comparable
```

---

## Trap 2: Type Switch Doesn't Work Directly on Type Parameters

**The trap:** "I wanted to check if `T` is a string inside my generic function."

**The problem:** You cannot type-switch directly on a type parameter value. The compiler doesn't allow it.

```go
// COMPILE ERROR: cannot use type switch on type parameter value
func Process[T any](v T) string {
    switch v.(type) {  // ERROR: cannot type switch on non-interface type T
    case int:
        return "int"
    case string:
        return "string"
    }
    return "other"
}

// FIX: convert to interface{} (or any) first
func Process[T any](v T) string {
    switch any(v).(type) {  // OK: convert to any first
    case int:
        return fmt.Sprintf("int: %d", any(v).(int))
    case string:
        return fmt.Sprintf("string: %s", any(v).(string))
    default:
        return fmt.Sprintf("other: %T", v)
    }
}
```

**But there's a deeper issue:** If you find yourself type-switching on a generic type parameter, it's often a sign you should be using an interface instead of generics. The point of generics is to write code that works uniformly for all types — if you need type-specific behavior, interfaces with methods are the right tool.

---

## Trap 3: Generics Don't Replace Interfaces

**The trap:** "Now that Go has generics, should I replace all my interfaces with type parameters?"

**The nuance:** No. They serve different purposes.

```go
// INTERFACE: when you need runtime polymorphism / extensibility
// Any type can implement io.Reader — you don't know all types upfront
type Reader interface {
    Read(p []byte) (n int, err error)
}
func Process(r io.Reader) { /* works with any Reader */ }

// GENERICS: when you want type-safe containers/algorithms over known type shapes
type Stack[T any] struct { items []T } // type-safe stack for any T
func Map[T, U any](s []T, f func(T) U) []U { /* ... */ }

// WRONG: replacing a behavioral interface with generics
// If you only have one concrete type, generics add no value
type HTTPClient[T any] interface {
    Do(req T) (*http.Response, error)
}
// This is overcomplicated — just use a function or interface{}

// RIGHT: generics for data structure parameterization
type Result[T any] struct { value T; err error }
// Now you can have Result[int], Result[User], Result[[]byte]
```

**Rule of thumb:**
- Generics: type-safe containers, utility functions (Map/Filter/Reduce)
- Interfaces: behavioral abstractions, plugin points, runtime polymorphism

---

## Trap 4: Methods Cannot Have Additional Type Parameters

**The trap:** "I want to add a `Map[U]` method to my generic list type."

**The problem:** In Go, methods on a generic type can only use the type parameters already declared on the type. They cannot introduce new type parameters.

```go
type List[T any] struct {
    items []T
}

// COMPILE ERROR: methods cannot have type parameters
func (l List[T]) Map[U any](fn func(T) U) List[U] {  // ERROR!
    // ...
}

// FIX 1: standalone function (most idiomatic)
func MapList[T, U any](l List[T], fn func(T) U) List[U] {
    result := List[U]{items: make([]U, len(l.items))}
    for i, v := range l.items {
        result.items[i] = fn(v)
    }
    return result
}

// FIX 2: redesign as a pipeline type (also works)
// Usage:
l := List[int]{items: []int{1, 2, 3}}
strs := MapList(l, strconv.Itoa) // List[string]
```

This is a known limitation as of Go 1.22. The Go team is aware and may address it in future versions.

---

## Trap 5: Generic Type Instantiation Creates New Types at Compile Time

**The trap:** "I assumed `Stack[int]` and `Stack[string]` are the same type."

**The reality:** They are distinct types. You cannot use them interchangeably.

```go
type Stack[T any] struct { items []T }

func processStack(s Stack[int]) { /* ... */ }

var intStack Stack[int]
var strStack Stack[string]

processStack(intStack) // OK
processStack(strStack) // COMPILE ERROR: cannot use Stack[string] as Stack[int]

// The types are genuinely different:
fmt.Printf("%T\n", intStack) // main.Stack[int]
fmt.Printf("%T\n", strStack) // main.Stack[string]

// You cannot store both in a []Stack[any]:
containers := []Stack[any]{} // ERROR: Stack[any] is a specific instantiation, not a supertype
```

If you need to store different instantiations in the same collection, use an interface:
```go
type Stacker interface {
    Len() int
    IsEmpty() bool
}

var containers []Stacker
containers = append(containers, &Stack[int]{})
containers = append(containers, &Stack[string]{}) // OK: both implement Stacker
```

---

## Trap 6: ~int vs int — Named Types Are Excluded Without Tilde

**The trap:** "I defined a constraint with `int` but my `UserID int` type doesn't satisfy it."

```go
// Without ~: only the exact type `int` satisfies this
type ExactInt interface { int }

type UserID int
type OrderID int

func sum[T ExactInt](a, b T) T { return a + b }
sum(1, 2)            // OK: untyped integer literal is int
sum(UserID(1), UserID(2)) // COMPILE ERROR: UserID does not implement ExactInt

// With ~: int AND any type with underlying type int
type AnyInt interface { ~int }

func sum[T AnyInt](a, b T) T { return a + b }
sum(UserID(1), UserID(2))   // OK: UserID's underlying type is int
sum(OrderID(1), OrderID(2)) // OK: same

// BUT: cannot mix types even if both satisfy ~int:
sum(UserID(1), OrderID(1)) // COMPILE ERROR: type mismatch (UserID vs OrderID)
```

This distinction matters a lot in practice. Library constraints like `constraints.Ordered` use `~` so they work with user-defined types. Custom constraints for domain types should almost always use `~`.

---

## Trap 7: Performance — Generics Share Code for Pointer Types (GCShape Stenciling)

**The trap:** "I thought generics meant each Stack[T] gets its own optimized machine code."

**The reality:** Go uses **GCShape stenciling**, not full monomorphization:
- Value types with different sizes get their own instantiation: `Stack[int]` ≠ `Stack[int32]`
- All pointer types share ONE instantiation: `Stack[*User]` == `Stack[*Product]` (same code!)
- Interface types share one instantiation: `Stack[io.Reader]` == `Stack[error]`

```go
// These compile to the SAME machine code (all are 8-byte pointers on 64-bit):
var _ = Stack[*User]{}
var _ = Stack[*Product]{}
var _ = Stack[*http.Request]{}

// These compile to DIFFERENT machine code (different sizes):
var _ = Stack[int]{}    // 8-byte values
var _ = Stack[int32]{}  // 4-byte values
var _ = Stack[bool]{}   // 1-byte values
```

**Practical implication:** Generics with pointer types won't make your code larger. The compilation also avoids the Java-style boxing performance penalty for value types. But it's not as aggressive as C++ templates (no per-type specialization beyond GC shape).

---

## Trap 8: Cannot Instantiate Generic Type With Itself Recursively in Some Cases

**The trap:** "I tried to create a recursive generic data structure and got a compile error."

```go
// This causes infinite type expansion — compile error:
type Tree[T any] struct {
    Value    T
    Children []Tree[T]  // This is fine — T is the same type parameter
}

// This is fine too:
t := Tree[int]{Value: 1, Children: []Tree[int]{
    {Value: 2},
    {Value: 3},
}}

// The problematic case: constraining with itself
type Ordered[T any] interface {
    Less(T) bool
}

// Recursive constraint (works in Go):
type Node[T Ordered[T]] struct { /* ... */ }
// But this requires T to implement Ordered[T] — complex constraint

// Simpler: use a comparison function instead
type BST[T any] struct {
    root *bstNode[T]
    less func(a, b T) bool
}

func NewBST[T any](less func(a, b T) bool) *BST[T] {
    return &BST[T]{less: less}
}

intTree := NewBST(func(a, b int) bool { return a < b })
```

---

## Trap 9: Generic Functions Can't Be Stored Directly in Variables of Specific Types

**The trap:** "I tried to assign a generic function to a typed variable and it fails."

```go
func Map[T, U any](slice []T, fn func(T) U) []U { /* ... */ }

// COMPILE ERROR: cannot use Map as value without type arguments
var f func([]int, func(int) string) []string = Map  // ERROR!

// FIX: wrap in a closure
var f func([]int, func(int) string) []string = func(s []int, fn func(int) string) []string {
    return Map(s, fn)
}
f([]int{1, 2, 3}, strconv.Itoa)  // OK

// Or: use a type alias for the instantiated function
type IntToStringMapper = func([]int, func(int) string) []string
var mapper IntToStringMapper = func(s []int, fn func(int) string) []string {
    return Map(s, fn)
}
```

Generic functions are not first-class values in Go — you cannot pass an uninstantiated generic function as a value. You must either instantiate it or wrap it.

---

## Trap 10: Zero Value for Type Parameter Requires `var zero T`

**The trap:** "I want to return a 'zero' value for T when my generic function fails."

**The problem:** You can't write `return nil` or `return 0` because `T` might be any type.

```go
// COMPILE ERROR: cannot use nil as type T
func Find[T any](slice []T, fn func(T) bool) T {
    for _, v := range slice {
        if fn(v) {
            return v
        }
    }
    return nil  // ERROR: nil is only valid for pointer, map, slice, chan, func, interface
}

// FIX: use var zero T (zero value of whatever T is)
func Find[T any](slice []T, fn func(T) bool) (T, bool) {
    for _, v := range slice {
        if fn(v) {
            return v, true
        }
    }
    var zero T  // zero value: 0 for int, "" for string, nil for pointer, etc.
    return zero, false
}

// Alternative: use *new(T) — equivalent but less readable
return *new(T), false

// Alternative with Go 1.18+: explicit zero
func zeroOf[T any]() T { var z T; return z }
```

This pattern (`var zero T`) appears everywhere in generic code. Get comfortable with it.
