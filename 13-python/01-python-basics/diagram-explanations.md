# Chapter 01 — Python Basics: Diagram Explanations

---

## Diagram 1: Python Type Hierarchy — Mutability Map

```
Python Built-in Types
│
├── Immutable (hashable, safe as dict key / set member)
│   ├── NoneType          → None
│   ├── bool              → True, False
│   ├── int               → 1, -5, 2**100
│   ├── float             → 3.14, 1e10
│   ├── complex           → 2+3j
│   ├── str               → "hello"
│   ├── bytes             → b"hello"
│   ├── tuple             → (1, 2, 3)       ← immutable CONTAINER
│   │                        (hashable only if all elements are hashable)
│   ├── frozenset         → frozenset({1, 2})
│   └── range             → range(10)
│
└── Mutable (NOT hashable, cannot be dict key / set member)
    ├── list              → [1, 2, 3]
    ├── dict              → {"a": 1}
    ├── set               → {1, 2, 3}
    └── bytearray         → bytearray(b"hello")
```

**Key rule:** If you try to use a mutable object as a dict key or set member,
you get `TypeError: unhashable type`. Hashability requires that the hash value
never changes, which immutability guarantees.

---

## Diagram 2: LEGB Scope Resolution — Visual Stack

```
┌─────────────────────────────────────────────────────┐
│  B — Built-in Scope                                 │
│  (provided by Python: len, print, range, int, ...)  │
│  ┌───────────────────────────────────────────────┐  │
│  │  G — Global Scope                             │  │
│  │  (module-level names, created with `=` at     │  │
│  │   top-level or declared `global` in func)     │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  E — Enclosing Scope                    │  │  │
│  │  │  (outer function's local variables,     │  │  │
│  │  │   relevant for nested functions)        │  │  │
│  │  │  ┌───────────────────────────────────┐  │  │  │
│  │  │  │  L — Local Scope                  │  │  │  │
│  │  │  │  (variables created inside the    │  │  │  │
│  │  │  │   current function)               │  │  │  │
│  │  │  └───────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘

Name lookup direction:  L → E → G → B  (innermost wins)
Modification direction: use `global` to write to G
                        use `nonlocal` to write to E
```

**Example trace:**
```python
x = 1        # G

def outer():
    x = 2    # E (for inner)
    def inner():
        x = 3        # L
        print(x)     # 3  ← L found, stop
    inner()
    print(x)         # 2  ← L not set here, E found
outer()
print(x)             # 1  ← G
```

---

## Diagram 3: Object Identity vs Value — Memory Model

```
After: a = [1, 2, 3]; b = a
                                    Heap
  Stack / Namespace                 ┌──────────────────────┐
  ┌────────────────┐                │  list object         │
  │  a  ───────────┼──────────────► │  id: 0x7f3a          │
  └────────────────┘                │  [1, 2, 3]           │
  ┌────────────────┐                └──────────────────────┘
  │  b  ───────────┼────────────────────────────────────────►(same object)
  └────────────────┘

a is b → True  (same id)
a == b → True  (same value)

─────────────────────────────────────────────────────────────

After: a = [1, 2, 3]; b = [1, 2, 3]

  Stack / Namespace                 Heap
  ┌────────────────┐                ┌──────────────────────┐
  │  a  ───────────┼──────────────► │  list object         │
  └────────────────┘                │  id: 0x7f3a          │
  ┌────────────────┐                │  [1, 2, 3]           │
  │  b  ───────────┼──────────────► └──────────────────────┘
  └────────────────┘                ┌──────────────────────┐
                                    │  list object         │
                                    │  id: 0x7f4b (differ) │
                                    │  [1, 2, 3]           │
                                    └──────────────────────┘

a is b → False (different id)
a == b → True  (same value)
```

---

## Diagram 4: Comprehension Forms and Their Outputs

```
Source iterable: range(1, 6)  →  1, 2, 3, 4, 5

┌─────────────────────────────────────────────────────────────────┐
│  FORM                SYNTAX                OUTPUT TYPE           │
├─────────────────────────────────────────────────────────────────┤
│  List                [x**2 for x in ...]   list (materialised)  │
│                      → [1, 4, 9, 16, 25]                        │
├─────────────────────────────────────────────────────────────────┤
│  Dict                {x: x**2 for x in …}  dict (materialised)  │
│                      → {1:1, 2:4, 3:9, …}                      │
├─────────────────────────────────────────────────────────────────┤
│  Set                 {x**2 for x in …}      set (materialised)  │
│                      → {1, 4, 9, 16, 25}   (order not guaranteed)│
├─────────────────────────────────────────────────────────────────┤
│  Generator           (x**2 for x in …)      generator (lazy)    │
│                      → <generator object>                        │
│                        values produced one at a time on demand   │
└─────────────────────────────────────────────────────────────────┘

Memory usage comparison:
  List comprehension   ████████████████  ~N * item_size bytes
  Generator expression █               ~112 bytes (just the object)

Execution flow for generator:
  caller: next(gen)
       ↓
  generator resumes, produces NEXT value
       ↓
  generator suspends (saves its state)
       ↓
  caller receives value

  Repeat until StopIteration is raised.
```

---

## Diagram 5: String Formatting Methods Comparison

```
┌─────────────────────────────────────────────────────────────────────┐
│  Method          Syntax                    Notes                     │
├─────────────────────────────────────────────────────────────────────┤
│  %-format        "Hello, %s" % name        Legacy; still used in     │
│                  "Val: %d" % 42            logging for lazy eval      │
├─────────────────────────────────────────────────────────────────────┤
│  str.format()    "Hello, {}".format(name)  More flexible; supports   │
│                  "{0} {1}".format(a, b)    positional & named args    │
├─────────────────────────────────────────────────────────────────────┤
│  f-string        f"Hello, {name}"          Fastest; evaluated at     │
│  (Python 3.6+)   f"{x:.2f}"               runtime; supports full    │
│                  f"{x=}" (3.8+ debug)      Python expressions        │
└─────────────────────────────────────────────────────────────────────┘

f-string feature tree:
  f"{...}"
      │
      ├── Variable reference:     f"{name}"
      ├── Expression:             f"{2 + 2}"
      ├── Method call:            f"{name.upper()}"
      ├── Format spec:            f"{pi:.4f}"    → '3.1416'
      │                           f"{n:>10}"     → right-align width 10
      │                           f"{n:0>5}"     → zero-padded width 5
      ├── Debug specifier (3.8):  f"{name=}"     → "name='Alice'"
      └── Nested f-string:        f"{'hello':>{width}}"
```

---

## Diagram 6: Unpacking and Extended Unpacking Flow

```
Basic unpacking:
  a, b, c = [10, 20, 30]
  ────┬──────┬──────┬────
      ↓      ↓      ↓
      a=10   b=20   c=30

Extended unpacking with *:
  a, *b, c = [10, 20, 30, 40, 50]
  ────┬────────────────────┬──────
      ↓                    ↓
      a=10     c=50
            *b captures the MIDDLE
            b = [20, 30, 40]

Rules:
  ┌────────────────────────────────────────────────────────┐
  │  Only ONE * allowed per assignment target              │
  │  * can appear at start, middle, or end                 │
  │  *b always collects as a LIST (even if empty)          │
  └────────────────────────────────────────────────────────┘

Position examples:
  *a, b     = [1, 2, 3]    → a=[1,2],  b=3
  a, *b     = [1, 2, 3]    → a=1,      b=[2,3]
  a, *b, c  = [1, 2, 3]    → a=1, b=[2], c=3
  a, *b, c  = [1, 2]       → a=1, b=[],  c=2

Function call unpacking (* and ** both allowed multiple times):
  [*list1, *list2]          → combine lists
  {**dict1, **dict2}        → merge dicts (right side wins on conflict)
  func(*args1, *args2)      → spread multiple iterables as positional args
```
