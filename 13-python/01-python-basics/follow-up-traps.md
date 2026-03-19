# Chapter 01 — Python Basics: Follow-Up Traps

---

## Trap 1: "Python is dynamically typed, so it has no types"

**What most people say:**
"Python doesn't care about types at all — you can put anything anywhere."

**Correct answer:**
Python is *dynamically typed* (types are checked at runtime, not compile time) but it
is also *strongly typed* — it will not silently coerce incompatible types. Try
`"3" + 3` and Python raises `TypeError`. In contrast, JavaScript (`"3" + 3 == "33"`)
is weakly typed. Python knows the type of every object at all times; it just doesn't
require you to declare types in advance.

```python
x = "3"
y = 3
x + y  # TypeError: can only concatenate str (not "int") to str

# contrast with implicit numeric widening (this IS allowed):
1 + 1.0   # → 2.0  (int promoted to float, a narrow coercion Python does allow)
```

---

## Trap 2: "is and == are interchangeable for comparing values"

**What most people say:**
"`is` is just a faster `==`, so I use them interchangeably."

**Correct answer:**
`==` calls `__eq__` and compares *value*. `is` compares *identity* (same object in
memory). They are not interchangeable. Small integers (-5 to 256) and interned strings
are cached, so `is` appears to work — but this is an implementation detail, not a
contract. Never use `is` to compare values; only use it to test for `None`, `True`,
and `False` singletons.

```python
a = 1000
b = 1000
a == b   # True
a is b   # False  (may be True in some REPL sessions due to compile-time optimization)

a = 256
b = 256
a is b   # True  — CPython caches integers -5..256

# Correct usage:
if x is None:   # good
if x == None:   # works but wrong style — a class can define __eq__ to equal None
```

---

## Trap 3: "Mutable default arguments are fine if I don't modify them"

**What most people say:**
"The mutable default argument bug only matters if the function actually mutates it,
so if I'm careful not to, it's fine."

**Correct answer:**
The deeper issue is that the default object is created *once* when the `def` statement
executes, not once per call. So the same object is shared across all calls. The correct
idiom uses `None` as the sentinel and creates a fresh object inside the body. This is
not just a mutation risk — it is a readability and API contract issue.

```python
# BUG: the list [] is created once at function definition time
def append_to(element, to=[]):
    to.append(element)
    return to

append_to(1)  # [1]
append_to(2)  # [1, 2]  ← NOT [2] !

# CORRECT:
def append_to(element, to=None):
    if to is None:
        to = []
    to.append(element)
    return to
```

---

## Trap 4: "The `else` on a for loop runs only if the loop completes without a break"

**What most people say (wrong direction):**
"The else runs if the loop body never executed" or "else means the loop was empty."

**Correct answer:**
`for...else` and `while...else` — the `else` block runs if the loop terminated
*normally* (exhausted its iterator or condition became False). It does NOT run if the
loop was exited via `break`. This is useful for "search and not found" patterns.

```python
def find_prime_factor(n, factors):
    for f in factors:
        if n % f == 0:
            print(f"Found factor: {f}")
            break
    else:
        # Only runs if no break occurred — meaning no factor was found
        print("No factor found")

find_prime_factor(10, [2, 3, 5])  # "Found factor: 2"
find_prime_factor(7, [2, 3, 5])   # "No factor found"
```

---

## Trap 5: "f-strings just do basic string interpolation"

**What most people say:**
"f-strings are just a cleaner way to write str.format()."

**Correct answer:**
f-strings evaluate arbitrary Python expressions, support format specs, call methods,
and since Python 3.8 support the `=` specifier for self-documenting debugging
expressions. They are evaluated at runtime, not at parse time, which means they can
reference closures and local variables.

```python
import math
x = 3.14159
name = "Alice"

# Format spec
f"{x:.2f}"           # '3.14'

# Expression
f"{2 ** 10}"         # '1024'

# Method call
f"{name.upper()}"    # 'ALICE'

# Debugging (3.8+)
f"{x=}"              # 'x=3.14159'
f"{math.pi=:.4f}"    # 'math.pi=3.1416'

# Nested format spec (dynamic width)
width = 10
f"{'hello':>{width}}"  # '     hello'
```

---

## Trap 6: "range() is just a list of numbers"

**What most people say:**
"`range(100)` creates a list of 100 numbers."

**Correct answer:**
`range` is a lazy sequence type — it computes elements on demand and stores only start,
stop, and step. It supports `len()`, indexing, slicing, and membership testing in O(1)
time. It is NOT a generator — you can iterate it multiple times without exhausting it.

```python
r = range(10**9)
len(r)       # 1000000000  — instant, no memory allocated
r[500]       # 500          — O(1) index
999999999 in r  # True      — O(1) membership test
list(r)      # MemoryError — now you've asked to materialise it

# range vs generator — range can be re-iterated:
r = range(5)
list(r)  # [0, 1, 2, 3, 4]
list(r)  # [0, 1, 2, 3, 4]  ← works again

g = (x for x in range(5))
list(g)  # [0, 1, 2, 3, 4]
list(g)  # []  ← generator is exhausted
```

---

## Trap 7: "Unpacking with * can appear anywhere and multiple times"

**What most people say:**
"You can use multiple `*` stars in an unpacking expression."

**Correct answer:**
In a single assignment target, only one `*` expression is allowed. However, in
function call arguments and in collection literals (Python 3.5+), multiple `*` and
`**` unpacking operators are allowed.

```python
# Assignment unpacking — only ONE * allowed:
a, *b, c = [1, 2, 3, 4, 5]   # OK: b = [2, 3, 4]
*a, *b = [1, 2, 3]             # SyntaxError

# Function call — multiple * allowed (Python 3.5+):
def f(a, b, c, d): return a + b + c + d
f(*[1, 2], *[3, 4])  # 10

# List/set/dict literals — multiple * and ** allowed:
combined = [*[1, 2], *[3, 4]]     # [1, 2, 3, 4]
merged = {**{"a": 1}, **{"b": 2}} # {'a': 1, 'b': 2}
```

---

## Trap 8: "None is falsy, so `if not x` catches None"

**What most people say:**
"`if not x` is the same as `if x is None`."

**Correct answer:**
`if not x` is `True` for ANY falsy value: `None`, `0`, `""`, `[]`, `{}`, `set()`,
`False`, `0.0`. Using `if not x` when you mean "check for None" creates subtle bugs
when valid values like `0`, `""`, or `[]` are legitimate inputs.

```python
def process(data=None):
    # BUG: rejects valid input of 0, "", or []
    if not data:
        data = get_default()

    # CORRECT: only handle the explicit None sentinel
    if data is None:
        data = get_default()

# Demonstrate the difference:
process(0)   # Bug version treats 0 as "not provided"
process([])  # Bug version treats [] as "not provided"
```

---

## Trap 9: "The walrus operator := is just assignment inside an expression"

**What most people say:**
"`:=` is the same as `=` but inside an expression, so I can use it anywhere."

**Correct answer:**
`:=` is subject to scoping rules that differ from `=`. In a list comprehension, a
regular variable is scoped to the comprehension; a walrus variable leaks into the
enclosing scope. This is a deliberate design choice that can surprise developers.
Also, `:=` has lower precedence than most operators and requires parentheses in some
contexts.

```python
# Walrus leaks out of comprehension scope:
result = [y := x * 2 for x in range(5)]
print(y)   # 8 — y is the LAST value assigned, available outside

# Regular comprehension variable does NOT leak:
result = [x * 2 for x in range(5)]
# print(x)  # NameError in Python 3

# Precedence trap — needs parens:
# if x := foo() == True:  # WRONG: parsed as (x := (foo() == True))
if (x := foo()) == True:  # correct grouping
    pass
```

---

## Trap 10: "bytes and str are interchangeable in Python 3"

**What most people say:**
"They both hold text, so you can mix them."

**Correct answer:**
In Python 3, `str` is a sequence of Unicode code points (text) and `bytes` is a
sequence of raw 8-bit values (binary data). They cannot be concatenated or compared
for equality without explicit encoding/decoding. This was a deliberate breaking change
from Python 2, which conflated them.

```python
s = "hello"
b = b"hello"

s == b          # False  (no implicit conversion)
s + b           # TypeError: can only concatenate str (not "bytes") to str

# Correct conversion:
s.encode("utf-8")        # b'hello'  — str  → bytes
b.decode("utf-8")        # 'hello'   — bytes → str

# Be explicit about encoding — don't assume ASCII:
"café".encode("utf-8")   # b'caf\xc3\xa9'  (2 bytes for é)
"café".encode("latin-1") # b'caf\xe9'       (1 byte for é)
```

---

## Trap 11: "Python integers are just 32-bit or 64-bit numbers"

**What most people say:**
"Python int is a 64-bit integer like in C."

**Correct answer:**
Python integers are arbitrary-precision (bignum) objects with no fixed upper bound.
CPython uses a variable-length array of "digits" (each 30 bits in CPython 3). This
means Python can compute `2 ** 1000` exactly, but very large integer operations are
slower than fixed-width operations because they involve heap-allocated objects.
Small integers (-5 to 256) are cached as singletons.

```python
2 ** 1000
# 107150860718626732094842504906000181056140481170553360744375038837
# 035105112493612249319837881569585812759467291755314682518714528569
# 231404359845775746985748360609... (301 digits)

import sys
sys.getsizeof(1)      # 28 bytes (CPython overhead for small int object)
sys.getsizeof(2**30)  # 32 bytes
sys.getsizeof(2**60)  # 36 bytes
sys.getsizeof(2**90)  # 40 bytes — grows with magnitude
```

---

## Trap 12: "String formatting with % is deprecated and should never be used"

**What most people say:**
"%-formatting is deprecated; always use f-strings."

**Correct answer:**
`%`-formatting is NOT formally deprecated in Python. It is still used in the `logging`
module intentionally — `logging.debug("value: %s", val)` passes the arguments lazily
so the string is only formatted if the log level is enabled. Using f-strings in
logging calls (`logging.debug(f"value: {val}")`) eagerly formats the string even when
the message will be discarded, wasting CPU.

```python
import logging
logging.basicConfig(level=logging.WARNING)

# BAD: string formatted even though DEBUG is disabled
logging.debug(f"Processing {expensive_computation()}")

# GOOD: string and call skipped entirely if DEBUG level not active
logging.debug("Processing %s", expensive_computation())
# Actually, even better — the function itself is still called above.
# Use a level check for truly expensive operations:
if logging.getLogger().isEnabledFor(logging.DEBUG):
    logging.debug("Processing %s", expensive_computation())
```
