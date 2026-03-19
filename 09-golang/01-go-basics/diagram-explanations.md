# Chapter 01 — Go Basics: Diagram Explanations

ASCII diagrams for visual understanding. Use these to explain Go concepts on a whiteboard.

---

## Diagram 1: Slice Header Structure and How append Works

### The Slice Header (3 fields)

```
 Slice variable 's'
┌──────────────────┐
│  Data (pointer)  │──────────────────────────┐
├──────────────────┤                           │
│  Len  (int)  = 3 │                           ▼
├──────────────────┤              Backing Array (in memory)
│  Cap  (int)  = 5 │         ┌────┬────┬────┬────┬────┐
└──────────────────┘         │ 10 │ 20 │ 30 │    │    │
                             └────┴────┴────┴────┴────┘
                              [0]  [1]  [2]  [3]  [4]
                              ▲              ▲         ▲
                           s[0]           s[len]     s[cap]
                           (accessible)            (reserved)
```

### append When len < cap (No Allocation)

```
Before: s = []int{10, 20, 30}  len=3, cap=5

Backing Array: [ 10 | 20 | 30 |    |    ]
                 0    1    2    3    4

s = append(s, 40)

After:  s has len=4, cap=5 (same backing array)

Backing Array: [ 10 | 20 | 30 | 40 |    ]
                 0    1    2    3    4
```

### append When len == cap (New Allocation)

```
Before: s = []int{10, 20, 30}  len=3, cap=3

Old Array: [ 10 | 20 | 30 ]
              0    1    2

s = append(s, 40)

Step 1: Allocate new array (typically 2x capacity)
New Array: [    |    |    |    |    |    ]

Step 2: Copy old elements
New Array: [ 10 | 20 | 30 |    |    |    ]

Step 3: Write new element
New Array: [ 10 | 20 | 30 | 40 |    |    ]

After: s has len=4, cap=6, pointing to NEW array
Old array is now unreferenced → eligible for GC
```

---

## Diagram 2: Array vs Slice Backing Array Sharing

```
Code:
arr := [6]int{1, 2, 3, 4, 5, 6}
s1 := arr[1:4]   // len=3, cap=5
s2 := arr[2:5]   // len=3, cap=4

Memory layout:

arr (array — owns the data):
┌───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │
└───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5

s1 (window starting at index 1):
      ┌───┬───┬───┐
      │ 2 │ 3 │ 4 │  len=3, cap=5
      └───┴───┴───┘
      ▲               ▲
   s1[0]           s1+cap

s2 (window starting at index 2):
          ┌───┬───┬───┐
          │ 3 │ 4 │ 5 │  len=3, cap=4
          └───┴───┴───┘
          ▲               ▲
       s2[0]           s2+cap

DANGER: s1[1] and s2[0] both refer to arr[2] (value 3)
Modifying s1[1] affects s2[0] and arr[2]!
```

---

## Diagram 3: Map Internal Structure (Hash Buckets)

```
map[string]int with 3 elements: "a"→1, "b"→2, "z"→9

Runtime hash function (seeded randomly per map):
  hash("a") = 0b...00101  → bucket 1
  hash("b") = 0b...10011  → bucket 3
  hash("z") = 0b...00101  → bucket 1 (collision!)

Bucket Array (8 buckets by default):
┌──────────┐
│ bucket 0 │ (empty)
├──────────┤
│ bucket 1 │──► ┌─────────────────────────────────┐
├──────────┤    │ tophash[0..7]  (high 8 bits)     │
│ bucket 2 │    ├─────────────────────────────────┤
├──────────┤    │ keys:   ["a", "z", "", "", ...]  │
│ bucket 3 │──► │ values: [ 1,   9,  0,  0, ...]   │
├──────────┤    │ overflow: nil                    │
│ ...      │    └─────────────────────────────────┘
└──────────┘

Each bucket holds up to 8 key-value pairs.
If more than 8 entries hash to the same bucket,
an overflow bucket is chained.

When load factor > ~6.5: bucket array doubles in size,
entries are incrementally moved to new buckets.
```

---

## Diagram 4: Stack Frame — Pointer vs Value Passing

```
Value passing (copy):

main goroutine stack                 modify() stack frame
┌─────────────────────┐             ┌─────────────────────┐
│ x  = 42             │──copy──────►│ x  = 42             │
│ (address: 0xc000)   │             │ (address: 0xd000)   │
└─────────────────────┘             │   x = 100           │
│                                   │ (changes local copy) │
│ After call: x still = 42          └─────────────────────┘
└────────────────────────────────────────────────────────

Pointer passing (reference):

main goroutine stack                 modify() stack frame
┌─────────────────────┐             ┌─────────────────────┐
│ x  = 42             │◄────────────│ p = &x = 0xc000     │
│ (address: 0xc000)   │  *p = 100  │ (address: 0xe000)   │
└─────────────────────┘             └─────────────────────┘
│
│ After call: x = 100 (modified through pointer)
└─────────────────────────────────────────────────────────

Slice passing (header copy, shared backing array):

main goroutine stack                 modify() stack frame
┌─────────────────────┐             ┌─────────────────────┐
│ s.Data = 0xf000     │             │ s.Data = 0xf000     │ (same!)
│ s.Len  = 3          │──copy──────►│ s.Len  = 3          │
│ s.Cap  = 5          │             │ s.Cap  = 5          │
└─────────────────────┘             └─────────────────────┘
           │                                   │
           └──────────────┬────────────────────┘
                          ▼
                 Backing Array (heap)
                ┌────┬────┬────┬────┬────┐
                │ 1  │ 2  │ 3  │    │    │
                └────┴────┴────┴────┴────┘
                Modifying s[i] affects BOTH — same array!
                append() may break the sharing (new array)
```

---

## Diagram 5: defer Stack (LIFO Execution Order)

```
Function execution:

func example() {
    fmt.Println("A")          ← executes now
    defer fmt.Println("D1")   ← pushed to defer stack
    fmt.Println("B")          ← executes now
    defer fmt.Println("D2")   ← pushed to defer stack
    fmt.Println("C")          ← executes now
    defer fmt.Println("D3")   ← pushed to defer stack
    // function returns
}

Defer stack grows during execution:
                                 ┌──────────┐
After defer D3: (top of stack)  │    D3    │  ← pushed last
                                ├──────────┤
                                │    D2    │
                                ├──────────┤
                                │    D1    │  ← pushed first
                                └──────────┘

Execution order on return (LIFO — pop from top):
                                 ┌──────────┐
1. Execute D3, pop              │    D2    │
                                ├──────────┤
2. Execute D2, pop              │    D1    │
                                ├──────────┤
3. Execute D1, pop              └──────────┘  (empty)

Final output:
A
B
C
D3
D2
D1
```

---

## Diagram 6: Go Program Structure (Packages → main → Imports)

```
Go Program Initialization Order
================================

Step 1: Dependency packages initialized first (topological order)
        ┌──────────────────────────────────────────────┐
        │           Package Dependency Graph           │
        │                                              │
        │  main ──depends──► myapp/db ──depends──► fmt │
        │    └──depends──► myapp/config               │
        │                                              │
        │  Initialization order:                       │
        │    fmt → myapp/db → myapp/config → main      │
        └──────────────────────────────────────────────┘

Step 2: For each package, in order:
        ┌────────────────────────────────────────────────────────┐
        │  a. Package-level var declarations evaluated           │
        │     var defaultTimeout = 30 * time.Second              │
        │     var db = connectDB() // evaluated at startup       │
        │                                                        │
        │  b. init() functions called (in source file order,     │
        │     files in alphabetical order within a package)      │
        │     func init() { log.SetFlags(log.LstdFlags) }        │
        └────────────────────────────────────────────────────────┘

Step 3: main() is called

Visual timeline:
═══════════════════════════════════════════════════════════

  [fmt init]──►[db vars]──►[db init()]──►[main vars]──►[main init()]──►[main()]

═══════════════════════════════════════════════════════════

Package file structure:
┌─────────────────────────────────────┐
│  myapp/                             │
│  ├── main.go        package main    │
│  ├── go.mod         module def      │
│  ├── go.sum         checksums       │
│  └── internal/                     │
│      ├── db/                        │
│      │   └── db.go  package db      │
│      └── config/                   │
│          └── config.go package cfg  │
└─────────────────────────────────────┘

Exported vs unexported:
┌──────────────────────────────────────────┐
│  Uppercase = Exported (public)           │
│  func Connect() { ... }   ← visible      │
│  type Config struct { ... } ← visible    │
│                                          │
│  Lowercase = Unexported (package-private)│
│  func connect() { ... }   ← hidden       │
│  type config struct { ... } ← hidden     │
└──────────────────────────────────────────┘
```

---

## Diagram 7: for Loop Variants in Go

```
Go's single loop keyword covers all iteration patterns:

1. C-style counted loop
   ┌─────────────────────────────────────┐
   │  for i := 0; i < n; i++ { ... }    │
   │       │         │       │           │
   │      init    condition  post        │
   └─────────────────────────────────────┘

2. While-style loop
   ┌─────────────────────────────────────┐
   │  for condition { ... }              │
   │  // equiv: for ; condition; { ... } │
   └─────────────────────────────────────┘

3. Infinite loop
   ┌─────────────────────────────────────┐
   │  for { ... }                        │
   │  // use break to exit               │
   └─────────────────────────────────────┘

4. Range over slice — gives index, value
   ┌─────────────────────────────────────────────────────┐
   │  for i, v := range slice { ... }                    │
   │       │  │                                          │
   │    index │                                          │
   │          value (COPY of element)                    │
   └─────────────────────────────────────────────────────┘

5. Range over map — gives key, value (random order)
   ┌─────────────────────────────────────────────────────┐
   │  for k, v := range m { ... }                        │
   └─────────────────────────────────────────────────────┘

6. Range over string — gives byte-index, rune
   ┌─────────────────────────────────────────────────────┐
   │  s := "Hello,世"                                     │
   │  for i, r := range s {                              │
   │       │  └── rune (Unicode code point)              │
   │       └─ byte index (NOT rune index!)               │
   │  }                                                  │
   │  // i jumps: 0,1,2,3,4,5,6,9 (世 is 3 bytes)       │
   └─────────────────────────────────────────────────────┘

7. Range over channel — reads until channel closed
   ┌─────────────────────────────────────────────────────┐
   │  for v := range ch { ... }                          │
   │  // blocks on each receive, exits when ch closed    │
   └─────────────────────────────────────────────────────┘
```
