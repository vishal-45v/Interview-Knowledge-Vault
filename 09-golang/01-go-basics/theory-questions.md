# Chapter 01 — Go Basics: Theory Questions

## Go vs Other Languages

**Q1. How does Go differ from Java and C++ in terms of memory management?**
Go uses a concurrent, tri-color mark-and-sweep garbage collector. Unlike Java's JVM GC, Go's GC is designed for low-latency applications and runs concurrently with the program. Unlike C++, there is no manual `delete` or RAII destructor pattern — Go's GC handles memory automatically. Go also avoids the overhead of a full virtual machine that Java requires.

**Q2. What is the key difference between goroutines and OS threads?**
OS threads are managed by the kernel and have a fixed stack size (typically 1–8 MB). Goroutines are managed by the Go runtime, start with a very small stack (~2 KB), and grow/shrink dynamically. You can spawn millions of goroutines; spawning millions of OS threads would exhaust system memory. Goroutines use an M:N scheduling model where M goroutines are multiplexed over N OS threads.

**Q3. Does Go support inheritance?**
No. Go deliberately excludes class-based inheritance. Instead, Go uses composition through struct embedding. If you embed a type, you get its fields and methods promoted to the outer type, but this is not inheritance — there is no `is-a` relationship, no method overriding, and no polymorphism via class hierarchy. Polymorphism in Go is achieved exclusively through interfaces.

**Q4. How does Go's compilation model differ from Java's?**
Go compiles directly to native machine code (like C/C++), not to bytecode for a virtual machine. This produces a single statically linked binary with no runtime dependencies (unless CGo is used). Java compiles to JVM bytecode and requires a JVM runtime. Go binaries start faster and have lower memory overhead for simple services.

**Q5. What is Go's stance on exceptions?**
Go has no exceptions. Errors are values — functions return an `error` as an explicit return value, and callers must handle it. `panic` exists for truly unrecoverable situations (like index out of bounds), and `recover` can catch a panic within a deferred function, but this is not used for normal control flow.

---

## Basic Types

**Q6. What are Go's basic built-in types?**
- **Integer**: `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr`
- **Floating point**: `float32`, `float64`
- **Complex**: `complex64`, `complex128`
- **Boolean**: `bool`
- **String**: `string` (immutable, UTF-8 encoded sequence of bytes)
- **Byte**: `byte` (alias for `uint8`)
- **Rune**: `rune` (alias for `int32`, represents a Unicode code point)

**Q7. What is the difference between `byte` and `rune`?**
`byte` is an alias for `uint8` and represents a single byte of data — useful for binary data and ASCII characters. `rune` is an alias for `int32` and represents a single Unicode code point. For example, the Chinese character `中` is one rune but three bytes in UTF-8. When iterating over a string with `range`, you get runes; when indexing with `s[i]`, you get bytes.

**Q8. What is the size of an `int` in Go?**
The size of `int` is platform-dependent: 32 bits on a 32-bit system and 64 bits on a 64-bit system. If you need a guaranteed size, use `int32` or `int64` explicitly.

**Q9. What is the zero value of basic types in Go?**
- `int`, `float64`, etc.: `0`
- `bool`: `false`
- `string`: `""` (empty string)
- `pointer`, `slice`, `map`, `channel`, `function`, `interface`: `nil`

**Q10. Can you compare different numeric types directly in Go?**
No. Go requires explicit type conversion. `int32(x) == int64(y)` would not compile — you must convert: `int64(x) == y`. This is a deliberate safety feature to prevent implicit data loss.

---

## Variables and Declaration

**Q11. What are the different ways to declare a variable in Go?**
```go
// Long form — explicit type, usable at package level
var x int = 10

// Long form — type inferred
var y = 10

// Long form — zero value
var z int

// Short declaration — only inside functions
a := 10

// Multiple
var b, c int = 1, 2
d, e := 3, 4
```

**Q12. What is the short variable declaration operator `:=` and where can it be used?**
`:=` is a combined declaration and assignment. It declares a new variable and infers the type from the right-hand side. It can only be used inside function bodies, not at the package level. At least one variable on the left side must be new — if all variables already exist, it's a compile error (except in cases where one variable is new).

**Q13. What is the zero value principle in Go?**
Every variable in Go is initialized to its zero value when declared without an explicit value. This eliminates uninitialized memory bugs. `var x int` gives `x == 0`, `var s string` gives `s == ""`, `var p *int` gives `p == nil`. This design choice makes Go programs more predictable.

**Q14. What is the blank identifier `_` in Go?**
The blank identifier `_` is used to discard values you don't need. Common uses:
```go
// Discard a return value
_, err := strconv.Atoi("42")

// Discard loop index
for _, v := range slice { ... }

// Ensure a type implements an interface at compile time
var _ io.Reader = (*MyReader)(nil)

// Execute init side effects of a package
import _ "net/http/pprof"
```

**Q15. What is the difference between `var x int` and `x := 0`?**
They both create an `int` with value `0`. The difference is scoping and syntax rules: `var x int` can appear at package level or inside a function; `x := 0` can only appear inside a function. In practice, inside functions, `:=` is preferred for brevity.

---

## Constants and iota

**Q16. What is `iota` in Go?**
`iota` is a built-in constant generator that starts at 0 and increments by 1 for each constant in a `const` block. It resets to 0 at each new `const` block.

```go
type Direction int

const (
    North Direction = iota // 0
    East                   // 1
    South                  // 2
    West                   // 3
)

// Skip a value using blank identifier
const (
    _  = iota // 0, discarded
    KB = 1 << (10 * iota) // 1 << 10 = 1024
    MB                    // 1 << 20
    GB                    // 1 << 30
)
```

**Q17. What is the difference between a constant and a variable in Go?**
Constants are evaluated at compile time, must be assigned a literal or constant expression, cannot be addressed (no `&const`), and cannot be of type slice, map, or struct. Variables are allocated at runtime (or on the stack) and can hold any type. Constants also have an "untyped" form that allows implicit conversion: an untyped integer constant `42` can be assigned to any integer type.

---

## Arrays and Slices

**Q18. What is the difference between an array and a slice in Go?**
An **array** has a fixed length that is part of its type: `[3]int` and `[4]int` are different types. Arrays are value types — assigning one array to another copies all elements. A **slice** is a dynamic, variable-length view into an underlying array. Slices are reference types — assigning a slice copies only the header (pointer, length, capacity), not the data.

**Q19. What does a slice header contain?**
A slice header is a three-field struct:
```
┌─────────┬─────┬─────┐
│ pointer │ len │ cap │
└─────────┴─────┴─────┘
```
- `pointer`: points to the first element of the underlying array the slice references
- `len`: number of elements accessible via the slice
- `cap`: number of elements from the pointer to the end of the underlying array

**Q20. How does `append` work in Go?**
`append` adds elements to a slice. If `len < cap`, it writes into the existing backing array and returns a new slice header with incremented `len`. If `len == cap`, a new, larger backing array is allocated (typically 2x capacity for small slices, ~1.25x for large ones), elements are copied, and a new header is returned. This means appending can create a new backing array, so always use the returned value: `s = append(s, elem)`.

**Q21. How do you create a slice in Go?**
```go
// Slice literal
s := []int{1, 2, 3}

// make — specify len and optional cap
s := make([]int, 5)        // len=5, cap=5
s := make([]int, 3, 10)    // len=3, cap=10

// Slice of an array
arr := [5]int{1, 2, 3, 4, 5}
s := arr[1:3] // elements 1 and 2

// nil slice (zero value)
var s []int // s == nil, len=0, cap=0
```

**Q22. What is the difference between `len` and `cap` on a slice?**
`len(s)` returns the number of elements currently in the slice. `cap(s)` returns the total number of elements in the underlying array starting from the slice's pointer. You can re-slice up to capacity: `s = s[:cap(s)]`.

---

## Maps

**Q23. How do you create a map in Go?**
```go
// Map literal
m := map[string]int{"a": 1, "b": 2}

// make
m := make(map[string]int)

// nil map — DO NOT write to this
var m map[string]int // m == nil
```

**Q24. What happens if you read from a nil map? What about write?**
Reading from a nil map is safe and returns the zero value of the value type. Writing to a nil map causes a runtime panic: `assignment to entry in nil map`. Always initialize maps with `make` or a literal before writing.

**Q25. How do you check if a key exists in a map?**
```go
v, ok := m["key"]
if ok {
    // key exists, v has the value
}
// v is the zero value of the value type if key doesn't exist
```

**Q26. How do you delete a key from a map?**
```go
delete(m, "key")
// Safe even if the key doesn't exist — no panic
```

**Q27. Is map iteration order guaranteed in Go?**
No. Map iteration order in Go is intentionally randomized. The Go team added this randomization deliberately (around Go 1.0) to prevent developers from depending on an implementation detail. If you need sorted iteration, extract the keys, sort them, then iterate.

**Q28. What types can be used as map keys?**
Any comparable type: booleans, integers, floats, complex numbers, strings, pointers, channels, interfaces, arrays (if element type is comparable), and structs (if all fields are comparable). Slices, maps, and functions cannot be map keys.

---

## Pointers

**Q29. What is a pointer in Go?**
A pointer is a variable that holds the memory address of another variable. `&x` gives the address of `x`. `*p` dereferences a pointer `p` to get the value it points to.

```go
x := 42
p := &x   // p is of type *int
fmt.Println(*p) // 42
*p = 100        // modifies x
fmt.Println(x)  // 100
```

**Q30. What is the difference between a pointer receiver and a value receiver on a method?**
A **value receiver** receives a copy of the struct. Modifications don't affect the original. A **pointer receiver** receives a pointer to the struct. Modifications affect the original. Use pointer receivers when: the method needs to modify the struct, the struct is large (avoid copying), or consistency requires it (if any method has a pointer receiver, make them all pointer receivers).

```go
type Counter struct{ count int }

func (c Counter) Get() int    { return c.count }         // value receiver
func (c *Counter) Inc()       { c.count++ }              // pointer receiver
```

---

## Functions

**Q31. Does Go support multiple return values?**
Yes. Multiple return values are idiomatic Go, primarily used for returning a result and an error together.
```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 2)
```

**Q32. What are variadic functions in Go?**
A variadic function accepts zero or more arguments of a specified type. The variadic parameter must be the last parameter. Inside the function it behaves as a slice.
```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

sum(1, 2, 3)
nums := []int{1, 2, 3}
sum(nums...) // spread a slice
```

**Q33. What is `defer` and when does it execute?**
`defer` schedules a function call to run when the surrounding function returns (normally or via panic). Deferred calls are pushed onto a stack and execute in LIFO (last-in, first-out) order. Common uses: closing files, releasing locks, measuring elapsed time.
```go
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // runs when readFile returns

    // ... use f
    return nil
}
```

**Q34. What are named return values in Go?**
Named return values are given names in the function signature and are pre-initialized to zero values. A `return` without arguments (naked return) returns the current values of the named returns.
```go
func minMax(nums []int) (min, max int) {
    min, max = nums[0], nums[0]
    for _, n := range nums[1:] {
        if n < min { min = n }
        if n > max { max = n }
    }
    return // naked return — returns min and max
}
```
Named returns improve readability for short functions but can cause confusion in longer ones. Naked returns in long functions are considered bad practice.

---

## Packages and Program Structure

**Q35. What is the `init` function in Go?**
`init` is a special function that runs automatically after all variable declarations in a package are initialized, but before `main`. A package can have multiple `init` functions (even in the same file). `init` cannot be called explicitly or referenced. Use it for package-level setup like registering drivers or initializing global state.

**Q36. What is the execution order when a Go program starts?**
1. All imported packages are initialized recursively (dependencies first)
2. Package-level `var` declarations are evaluated
3. `init()` functions run (in the order they appear, across files alphabetically)
4. `main()` runs

**Q37. What is the rule for exported names in Go?**
A name is exported (visible outside its package) if it begins with an uppercase letter. `Printf` is exported; `printf` is not. This applies to functions, types, variables, constants, struct fields, and interface methods.

---

## Control Flow

**Q38. What is the only looping construct in Go?**
Go has only one looping keyword: `for`. It covers three patterns:
```go
// Traditional C-style for
for i := 0; i < 10; i++ { }

// While-style
for condition { }

// Infinite loop
for { }

// Range over slice, string, map, channel
for i, v := range slice { }
for k, v := range myMap { }
for i, r := range "hello" { } // i=byte index, r=rune
```

**Q39. How does Go's `switch` differ from C/Java?**
In Go, cases do not fall through by default (no need for `break`). Each case is implicitly followed by a `break`. To fall through explicitly, use the `fallthrough` keyword. Switch cases can be expressions, not just constants. Go also supports a `type switch` to check the dynamic type of an interface.

```go
// Expression switch
switch x {
case 1, 2:
    fmt.Println("one or two")
case 3:
    fmt.Println("three")
default:
    fmt.Println("other")
}

// Type switch
switch v := i.(type) {
case int:
    fmt.Printf("int: %d\n", v)
case string:
    fmt.Printf("string: %s\n", v)
default:
    fmt.Printf("unknown type: %T\n", v)
}
```

---

## Type System

**Q40. How does type conversion work in Go?**
Go requires explicit type conversions between different types — there are no implicit conversions. Use `T(x)` syntax.
```go
var i int = 42
var f float64 = float64(i) // explicit conversion
var u uint = uint(f)

// String conversions
s := string(65)          // "A" (rune to string)
s := fmt.Sprintf("%d", i) // integer to string representation
n, _ := strconv.Atoi("42") // string to int
```

**Q41. What is a named type in Go?**
A named type is a user-defined type based on an existing type. Even if the underlying type is the same, named types are distinct and require explicit conversion.
```go
type Celsius float64
type Fahrenheit float64

c := Celsius(100.0)
f := Fahrenheit(212.0)
// c + f would be a compile error
// Must convert explicitly
```

---

## Closures

**Q42. What is a closure in Go?**
A closure is a function value that captures variables from its surrounding scope. The closure can read and modify those variables even after the surrounding function has returned.

```go
func makeCounter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

c := makeCounter()
fmt.Println(c()) // 1
fmt.Println(c()) // 2
fmt.Println(c()) // 3
```

**Q43. What is the loop variable capture problem with closures?**
Before Go 1.22, loop variables were shared across all iterations. Closures created in the loop would all capture the same variable and see its final value. Go 1.22 fixed this by making each iteration create a new variable.

```go
// Pre-Go 1.22 problem
funcs := make([]func(), 3)
for i := 0; i < 3; i++ {
    funcs[i] = func() { fmt.Println(i) }
}
// Calling funcs[0](), funcs[1](), funcs[2]() all print 3

// Fix (pre-1.22): create a new variable per iteration
for i := 0; i < 3; i++ {
    i := i // shadows outer i
    funcs[i] = func() { fmt.Println(i) }
}
```

---

## Structs

**Q44. How do you define and use a struct in Go?**
```go
type Person struct {
    Name string
    Age  int
    email string // unexported field
}

// Struct literal (positional — fragile, avoid)
p1 := Person{"Alice", 30, "a@example.com"}

// Struct literal (named — preferred)
p2 := Person{Name: "Bob", Age: 25}

// Zero value
var p3 Person // Name="", Age=0, email=""
```

**Q45. What is an anonymous struct and when is it useful?**
An anonymous struct is a struct defined inline without a named type. Useful for one-off data grouping, JSON test data, and grouping related variables.
```go
config := struct {
    Host string
    Port int
}{
    Host: "localhost",
    Port: 8080,
}

// Useful in table-driven tests
tests := []struct {
    input    string
    expected int
}{
    {"42", 42},
    {"0", 0},
}
```

**Q46. Can structs be compared with `==` in Go?**
Structs are comparable with `==` if all their fields are comparable types. Two structs are equal if all their fields are equal. Structs containing slices, maps, or functions cannot be compared with `==`.

```go
type Point struct{ X, Y int }
p1 := Point{1, 2}
p2 := Point{1, 2}
fmt.Println(p1 == p2) // true

type WithSlice struct{ Data []int }
// w1 == w2 would be a compile error
```

---

**Total: 46 theory questions covering all fundamental Go concepts.**
