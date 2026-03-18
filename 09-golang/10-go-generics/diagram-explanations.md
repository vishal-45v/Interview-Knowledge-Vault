# Go Generics — Diagram Explanations

---

## Diagram 1: Generic Function Syntax Anatomy

```
Complete generic function with two type parameters:

func  Map  [  T  ,    U    any  ]  (  slice  []T  ,  fn  func(T) U  )  []U
│     │    │  │       │    │    │  │  │           │      │           │  │
│     │    │  │       │    │    │  │  │           │      │           │  └─ return type
│     │    │  │       │    │    │  │  │           │      └───────────── transform fn
│     │    │  │       │    │    │  │  └───────────────────────────────── first param
│     │    │  │       │    │    │  └── parameter list starts
│     │    │  │       │    └────────── constraint (both T and U must satisfy)
│     │    │  │       └────────────── second type parameter
│     │    │  └────────────────────── first type parameter
│     │    └───────────────────────── type parameter list starts
│     └────────────────────────────── function name
└──────────────────────────────────── keyword

COMPONENTS:
┌─────────────────────────────────────────────────────────────────────┐
│ Type Parameter List: [T, U any]                                     │
│   • Square brackets distinguish from regular parameters             │
│   • Multiple params separated by commas                             │
│   • Constraint appears after the last param name in the group       │
│   • [T any, U comparable] — different constraints per param         │
│   • [T, U any] — shorthand: same constraint for T and U             │
└─────────────────────────────────────────────────────────────────────┘

MORE EXAMPLES:

// Single type param, built-in constraint:
func Contains[T comparable](slice []T, item T) bool
              └─────────────┘
              one param, comparable constraint

// Generic type with type param in receiver:
func (s *Stack[T]) Push(item T)
      └────────┘
      receiver uses the Stack's type param T
      (no new type params in method — this is required)

// Inline constraint (unnamed interface):
func Add[T interface{ ~int | ~float64 }](a, b T) T
         └──────────────────────────────┘
         inline anonymous constraint

// Multiple constraints:
func Copy[T any, K comparable](dst map[K]T, src map[K]T)
         ┌──┐  ┌────────────┐
         T=any K=comparable
```

---

## Diagram 2: Type Constraint Hierarchy

```
BUILT-IN CONSTRAINTS (no import needed):

      any (= interface{})
       │
       │  Every type satisfies any
       │
       ├─── comparable
       │     Satisfies: int, float, string, bool, pointer,
       │                array (if elements comparable),
       │                struct (if all fields comparable)
       │     Does NOT: slice, map, function, interface
       │
       └─── (no further built-in narrowing)

GOLANG.ORG/X/EXP/CONSTRAINTS (and cmp package, Go 1.21):

      any
       └── comparable
            └── constraints.Integer / cmp.Ordered (partial overlap)
                 │
                 ├── Signed: ~int | ~int8 | ~int16 | ~int32 | ~int64
                 │
                 └── Unsigned: ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr

      constraints.Float: ~float32 | ~float64

      constraints.Ordered (= cmp.Ordered in Go 1.21):
          ~int | ~int8 | ~int16 | ~int32 | ~int64 |
          ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
          ~float32 | ~float64 | ~string

CUSTOM CONSTRAINT EXAMPLES:

      type Numeric interface {
          ~int | ~int8 | ~int16 | ~int32 | ~int64 |
          ~float32 | ~float64
      }
      // Satisfies: int, float64, MyInt (where type MyInt int)
      // Does NOT: string, bool, struct

      type Stringish interface { ~string }
      // Satisfies: string, type Name string, type Email string
      // Does NOT: []byte, int

VENN DIAGRAM OF COMMON CONSTRAINTS:
   ┌─────────────────────────────────────────────────────┐
   │  any (all types)                                    │
   │  ┌───────────────────────────────────────────────┐  │
   │  │  comparable (supports ==, !=)                  │  │
   │  │  ┌─────────────────────────────────────────┐  │  │
   │  │  │  Ordered (supports <, >, <=, >=)         │  │  │
   │  │  │  ints, floats, strings                   │  │  │
   │  │  └─────────────────────────────────────────┘  │  │
   │  │  + pointers, channels, arrays, structs         │  │
   │  └───────────────────────────────────────────────┘  │
   │  + slices, maps, functions (not comparable)         │
   └─────────────────────────────────────────────────────┘
```

---

## Diagram 3: Generic Stack[T] Structure

```
Stack[int] (instantiated for int):

  Stack[int] struct
  ┌──────────────────────────────────────────────┐
  │  items  []int                                │
  │         ┌──────────────────────────────────┐ │
  │         │ len: 3  cap: 4                   │ │
  │         │ ptr ──────────────────────────►  │ │
  │         └──────────────────────────────────┘ │
  └──────────────────────────────────────────────┘
                                     │
                                     ▼
                     ┌────┬────┬────┬────┐
                     │ 10 │ 20 │ 30 │    │
                     └────┴────┴────┴────┘
                       [0]  [1]  [2]  [3] (empty slot)

  Push(40):  append 40 at index 3
  Pop():     return 30, shrink to len=2
  Peek():    return 30, no change

METHODS ON STACK[T]:

  (s *Stack[T]) Push(item T)
                     ────┬──
                         └── T is whatever type the stack was instantiated with

  (s *Stack[T]) Pop() (T, bool)
                       ─┬─
                         └── returns a T (zero value if empty)

  // The zero value of T:
  var zero T  // 0 for int, "" for string, nil for *User, etc.


Stack[string] (same code structure, different type):

  Stack[string] struct
  ┌──────────────────────────────────────────────────┐
  │  items  []string                                 │
  │         ┌────────────────────────────────────┐   │
  │         │ ptr → ["hello", "world", "!"]      │   │
  │         └────────────────────────────────────┘   │
  └──────────────────────────────────────────────────┘

TWO SEPARATE TYPES AT COMPILE TIME:

  typeof(Stack[int])    ≠  typeof(Stack[string])
  Stack[int]{}             Stack[string]{}
  can NOT be used together without explicit conversion
```

---

## Diagram 4: Type Inference Flow

```
How the compiler deduces T from function arguments:

func Map[T, U any](slice []T, fn func(T) U) []U

Call site: Map([]int{1, 2, 3}, strconv.Itoa)
                │                  │
                │                  │
  COMPILER ANALYSIS:               │
                │                  │
  Step 1: Look at arg 1:          │
          []int{1, 2, 3}           │
          matches param: []T       │
          → T = int                │
                                   │
  Step 2: Look at arg 2:          │
          strconv.Itoa             │
          type: func(int) string   │
          matches param: func(T) U │
          T is already int         │
          → U = string             │
                │
  RESOLVED: T = int, U = string

  Generates call to: Map[int, string]([]int{1,2,3}, strconv.Itoa)


INFERENCE FAILS (must specify explicitly):

  Case 1: No arguments to infer from
  ┌──────────────────────────────────────────────────────────────┐
  │  type Stack[T any] struct { items []T }                      │
  │  s := Stack{}         // ERROR: can't infer T — no arguments! │
  │  s := Stack[int]{}    // CORRECT: explicit                    │
  └──────────────────────────────────────────────────────────────┘

  Case 2: Ambiguous inference
  ┌──────────────────────────────────────────────────────────────┐
  │  func Convert[T, U any](v T) U { ... }                       │
  │  x := Convert(42)     // ERROR: U cannot be inferred         │
  │  x := Convert[int, string](42) // CORRECT: explicit          │
  └──────────────────────────────────────────────────────────────┘

  Case 3: Return type inference (Go can infer T from arguments but not always from return)
  ┌──────────────────────────────────────────────────────────────┐
  │  func New[T any]() T { var z T; return z }                   │
  │  x := New()        // ERROR: T can't be inferred             │
  │  x := New[int]()   // CORRECT                                │
  └──────────────────────────────────────────────────────────────┘
```

---

## Diagram 5: Go Generics vs Java Generics (Stenciling vs Type Erasure)

```
JAVA: Type Erasure

Source code:                     Bytecode (runtime):
List<Integer>     ─────────►    List<Object>
List<String>      ─────────►    List<Object>  (same class!)
List<Double>      ─────────►    List<Object>

  nums.add(42)
       │
       ▼
  autoboxing: int 42 → new Integer(42)
              └── HEAP ALLOCATION! 16 bytes per integer
       │
       ▼
  Object[] backing array
  ┌──────┬──────┬──────┬──────┐
  │ ref  │ ref  │ ref  │ ref  │  ← each slot is a 8-byte pointer
  └──────┴──────┴──────┴──────┘
     │       │       │
     ▼       ▼       ▼
  [Obj]   [Obj]   [Obj]       ← separate heap objects for each int!

  int x = nums.get(0)
       │
       ▼
  unboxing: Integer.intValue()  ← extra method call


GO: GCShape Stenciling

Source code:        Compiled code (per GC shape):
Stack[int]     →   stencil_valueN_8byte    (64-bit int operations)
Stack[int32]   →   stencil_valueN_4byte    (32-bit int operations)
Stack[*User]   →   stencil_ptr             (pointer operations)  ┐ SHARED!
Stack[*Product]→   stencil_ptr             (pointer operations)  ┘
Stack[io.Reader]→  stencil_iface           (interface operations)

  s.Push(42)  // Stack[int]
       │
       ▼
  s.items = append(s.items, 42)  ← 42 stored directly

  int[] backing array
  ┌────┬────┬────┬────┐
  │ 42 │ 17 │ 99 │    │  ← actual int values, no boxing!
  └────┴────┴────┴────┘


PERFORMANCE COMPARISON (100M operations):
┌───────────────────┬──────────────────┬──────────────────────────┐
│ Operation         │ Java List<Int>   │ Go Stack[int]             │
├───────────────────┼──────────────────┼──────────────────────────┤
│ Push int          │ ~20ns (boxing)   │ ~1ns (direct)             │
│ Pop int           │ ~15ns (unboxing) │ ~1ns (direct)             │
│ Memory / element  │ ~16 bytes        │ 8 bytes                   │
│ GC pressure       │ high (objects)   │ low (contiguous ints)     │
└───────────────────┴──────────────────┴──────────────────────────┘

NOTE: For pointer types, Go and Java have similar performance
      (both just store pointers). The difference is for value types
      (int, float, struct) where Go avoids boxing entirely.
```
