# Go Generics — Analogy Explanations

---

## Type Parameter: A Blank Label on a Box

Imagine a shipping warehouse with a special type of box called a "CoolBox." The factory that makes CoolBoxes leaves a blank label on the side, with instructions: "Write what goes inside when you order this box."

- When you order for electronics, you write "ELECTRONICS" on the label → you get a CoolBox for electronics
- When you order for books, you write "BOOKS" → you get a CoolBox for books
- The box works the same way regardless of what's inside (same compartments, same lock mechanism)

A **type parameter** is that blank label:

```go
type Stack[T any] struct {  // T = blank label
    items []T
}

// "Fill in" the label when you use it:
intStack  := Stack[int]{}    // label says "INT"
strStack  := Stack[string]{} // label says "STRING"
userStack := Stack[User]{}   // label says "USER"
```

All three stacks work identically (Push, Pop, Peek) — only the type of thing stored differs. The blueprint (Stack definition) is written once. The specific instances are different boxes with different labels.

The compiler fills in the label at compile time, not runtime. This means if you try to push a string into an `intStack`, you get a compile error — the label is checked before the box is shipped.

---

## Type Constraint: A Job Requirement

A company posts a job description: "Must have at least 5 years of experience and be proficient in Python." Not just anyone can apply — only candidates meeting the requirements.

A **type constraint** is that job requirement for type parameters:

```go
// Job posting: "Must be sortable (implement <, >, <=, >=)"
type Ordered interface {
    ~int | ~float64 | ~string
}

// Function that only accepts "candidates" meeting the requirement:
func Max[T Ordered](a, b T) T {
    if a > b { return a }
    return b
}

Max(3, 4)          // int meets requirement ✓
Max(3.14, 2.71)    // float64 meets requirement ✓
Max("foo", "bar")  // string meets requirement ✓
Max(true, false)   // bool does NOT meet requirement ✗ — compile error
```

The constraint says "I need a type that can be compared with `<`." The compiler checks at compile time — if you pass an incompatible type, you get a compiler error, not a runtime panic. It's like HR screening candidates before the interview, not failing them on the first day of work.

Different jobs have different requirements:
- `any` = "no requirements, anyone can apply"
- `comparable` = "must be equality-testable (==)"
- `constraints.Ordered` = "must support ordering operators (<, >, <=, >=)"
- Your custom constraint = "must support these specific operations"

---

## Generic Function: A Universal Remote

A universal remote control works with any TV brand that supports HDMI-CEC. You don't need to know if it's a Samsung, LG, or Sony TV — as long as it "speaks HDMI-CEC," the remote can power it on, change the volume, and switch inputs.

```go
// The "HDMI-CEC" requirement (constraint):
type RemoteControllable interface {
    PowerOn() error
    SetVolume(level int) error
    SwitchInput(input string) error
}

// The universal remote (generic function):
func LaunchMovie[T RemoteControllable](device T, movie string) error {
    if err := device.PowerOn(); err != nil {
        return err
    }
    device.SetVolume(50)
    device.SwitchInput("HDMI-1")
    // play the movie...
    return nil
}

// Works with any device meeting the requirement:
var samsung SamsungTV
var lg LGTV
LaunchMovie(samsung, "Inception")   // ✓
LaunchMovie(lg, "Interstellar")     // ✓
// LaunchMovie(myToaster, "movie")  // ✗ compile error: toaster isn't RemoteControllable
```

Without generics: you'd write a `LaunchMovieForSamsung`, a `LaunchMovieForLG`, etc. With generics: one function, works for any compliant device, type-safe. The universal remote doesn't care which brand — it cares about the interface.

---

## comparable Constraint: Items With Serial Numbers

In a warehouse, you can sort items two ways:
1. Items with **serial numbers** (unique IDs) — you can check: "Is item #A1234 == item #B5678?" Comparing is easy: compare the serial numbers.
2. Items with **no serial number** (e.g., a bag of sand) — you can't just compare. "Is this bag of sand == that bag of sand?" There's no simple yes/no answer.

The `comparable` constraint says: "I only accept items with serial numbers (items that support `==` and `!=`)."

```go
// Items with "serial numbers" (comparable):
// int, string, bool, pointers, arrays, structs with comparable fields

// Items without "serial numbers" (not comparable):
// slices ([]int), maps (map[string]int), functions (func())

func SetAdd[T comparable](set map[T]struct{}, item T) {
    set[item] = struct{}{} // needs == to check if key exists
}

SetAdd(map[int]struct{}{}, 42)           // int has a serial number ✓
SetAdd(map[string]struct{}{}, "hi")      // string has a serial number ✓
// SetAdd(map[[]int]struct{}{}, []int{1}) // COMPILE ERROR: slice has no serial number ✗
```

If you try to use a slice as a map key: the warehouse can't find it. Is `[1,2,3]` the same as `[1,2,3]`? In Go's model, you can't directly compare slices with `==` — so slices don't qualify for `comparable`.

---

## The ~ Tilde Operator: Family Resemblance

Imagine you're organizing a library and you have a category rule: "Philosophy books go here."

- **Without ~**: Only books explicitly labeled "Philosophy" qualify. A book labeled "Eastern Philosophy" — even though it IS philosophy — doesn't go in the category.

- **With ~**: Any book whose fundamental subject (underlying type) is philosophy qualifies. "Eastern Philosophy," "Political Philosophy," "Applied Ethics" — all get included because their essence is philosophical.

```go
// Without ~: only the exact type string
type StrictString interface { string }

type Genre string
type BookTitle string

// func format[T StrictString](t T) — Genre and BookTitle DON'T qualify!

// With ~: string and any type built on string
type StringLike interface { ~string }

func format[T StringLike](t T) string {
    return strings.ToUpper(string(t))
}

format(Genre("Fiction"))              // OK — Genre's nature is string ✓
format(BookTitle("Being and Time"))   // OK — BookTitle's nature is string ✓
// format(42)                         // ERROR — int is not built on string ✗
```

In practical terms: you define domain-specific types like `UserID int` and `OrderID int` for type safety. Without `~`, neither would work in a generic integer function. With `~int`, both `UserID` and `OrderID` qualify because their underlying nature is integer — even though they're named differently to prevent accidental mixing.

```go
type UserID  int
type OrderID int

type AnyInt interface { ~int }

func addTwo[T AnyInt](a, b T) T { return a + b }

addTwo(UserID(10), UserID(20))   // OK: UserID's family = int ✓
addTwo(OrderID(5), OrderID(3))   // OK: OrderID's family = int ✓
// addTwo(UserID(1), OrderID(1)) // ERROR: type mismatch (still type-safe!)
```

---

## Go Generics vs Java Generics: Customized Production vs Generic Packaging

Think of a packaging factory:

**Java generics (type erasure):** The factory makes ONE standard-size box regardless of what's inside. Shipping a diamond ring? It goes in the same box as a watermelon — just wrapped in a generic "Object" wrapper. At delivery time, the recipient has to unwrap and check ("Is this a ring or a watermelon?"). Putting a small int in the box requires first putting it in a special Integer envelope (boxing — an extra heap allocation).

```java
List<Integer> nums = new ArrayList<>();
nums.add(42);  // 42 wrapped in Integer envelope — heap allocation
Integer n = nums.get(0);  // unwrap the envelope
int x = n;  // unbox back to primitive
```

**Go generics (GCShape stenciling):** The factory has a smart configurator. For each unique "shape" of item, it adjusts the production line:
- Items that are all 8-byte values get the "8-byte production line" (ints, int64s, float64s)
- Items that are all 4-byte values get the "4-byte production line" (int32, float32)
- Items that are pointers (all 8 bytes on 64-bit) share one "pointer production line" — `*User`, `*Product`, `*http.Request` all use the same machinery

```go
// Each size gets its own machinery:
Stack[int]    // 8-byte production line
Stack[int32]  // 4-byte production line

// All pointers share one line (same 8-byte size):
Stack[*User]    // pointer production line
Stack[*Product] // same pointer production line (shared!)

// No wrapping:
var s Stack[int]
s.Push(42) // 42 is stored directly as an int — no envelope, no heap allocation
```

The result: Go generics are faster than Java generics for numbers and structs because there's no boxing. A `Stack[int]` in Go works as fast as a hand-written `StackInt` — the factory produces the real item, not a wrapped generic substitute.
