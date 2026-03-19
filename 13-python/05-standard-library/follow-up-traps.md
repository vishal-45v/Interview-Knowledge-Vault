# Chapter 05 — Standard Library: Follow-Up Traps

---

## Trap 1: "defaultdict will raise KeyError like a regular dict"

**What most people say:**
"`defaultdict` handles missing keys but otherwise behaves like a regular dict."

**Correct answer:**
`defaultdict` automatically creates a new entry when you ACCESS a missing key via
`__getitem__` (i.e., `d[key]`). This means checking `key in d` does NOT create it,
but `d[key]` does — even if you are just reading. This can cause spurious keys to be
created by innocent-looking access patterns, bloating the dict unexpectedly.

```python
from collections import defaultdict

d = defaultdict(list)

# CREATES the key (might be unintentional):
value = d["missing_key"]    # → []  and "missing_key" is now IN d!
"missing_key" in d           # True — side effect of reading!

# Does NOT create the key:
"other_key" in d             # False — membership test doesn't trigger factory
d.get("other_key")           # None  — .get() doesn't trigger factory

# Practical danger:
d = defaultdict(int)
for key in some_keys:
    d[key] += 1    # intended

# Later, a typo:
if d["typ0_key"] > 0:   # creates "typ0_key": 0 in d — not the intent
    process(d["typ0_key"])
```

---

## Trap 2: "OrderedDict is redundant since Python 3.7"

**What most people say:**
"Since Python 3.7 dicts preserve insertion order, `OrderedDict` is legacy code."

**Correct answer:**
`OrderedDict` retains several unique behaviours that plain `dict` does not have:
(1) `move_to_end(key, last=True)` — move an item to either end, useful for LRU caches;
(2) equality comparison is ORDER-SENSITIVE for `OrderedDict` but not for plain `dict`;
(3) `popitem(last=True/False)` — pop from either end.

```python
from collections import OrderedDict

d1 = OrderedDict([("a", 1), ("b", 2)])
d2 = OrderedDict([("b", 2), ("a", 1)])
d1 == d2   # False — order matters for OrderedDict equality!

d1_plain = {"a": 1, "b": 2}
d2_plain = {"b": 2, "a": 1}
d1_plain == d2_plain   # True — order doesn't matter for plain dict equality

# LRU cache pattern with move_to_end:
cache = OrderedDict()
cache["page1"] = "content1"
cache["page2"] = "content2"
cache.move_to_end("page1")  # page1 is now the "most recently used"
cache.popitem(last=False)   # removes the LRU item (from front)
```

---

## Trap 3: "contextmanager generator can yield multiple times"

**What most people say:**
"A `@contextmanager` generator can yield multiple times if needed."

**Correct answer:**
A `@contextmanager` generator must yield EXACTLY ONCE. If it yields zero times,
you get a `RuntimeError`. If it yields more than once, you also get a `RuntimeError`.
The `yield` statement is the boundary between `__enter__` and `__exit__`.

```python
from contextlib import contextmanager

# WRONG — zero yields:
@contextmanager
def bad_context():
    setup()
    if condition:
        yield   # only sometimes yields → RuntimeError on some calls

# WRONG — two yields:
@contextmanager
def double_yield():
    yield "first"
    yield "second"   # RuntimeError: generator didn't stop after __exit__

# CORRECT — exactly one yield, always:
@contextmanager
def good_context():
    resource = setup()
    try:
        yield resource   # __enter__ return value
    except SomeError:
        handle_error()
        raise
    finally:
        teardown(resource)   # always runs — equivalent to __exit__
```

---

## Trap 4: "re.match() and re.search() are interchangeable"

**What most people say:**
"`re.match()` and `re.search()` both search for a pattern in a string."

**Correct answer:**
`re.match()` only matches at the **beginning** of the string. `re.search()` scans the
entire string and returns the first match anywhere. `re.fullmatch()` requires the
pattern to match the **entire** string.

```python
import re

text = "hello world"

re.match(r"world", text)     # None — "world" not at start
re.search(r"world", text)    # Match at position 6

re.match(r"hello", text)     # Match at position 0
re.search(r"hello", text)    # Match at position 0 — same here

# fullmatch:
re.fullmatch(r"\d+", "123")   # Match — entire string is digits
re.fullmatch(r"\d+", "123abc")# None — "abc" not matched

# Practical implication — validating a string:
def is_valid_id(s):
    return bool(re.match(r"\d{5}$", s))   # need $ anchor with match
    # OR: bool(re.fullmatch(r"\d{5}", s)) # cleaner and explicit

is_valid_id("12345")    # True
is_valid_id("12345xyz") # False — $ forces end-of-string
```

---

## Trap 5: "Logging with f-strings is always fine"

**What most people say:**
"Using f-strings in logging is the modern way and equivalent to % formatting."

**Correct answer:**
f-strings are evaluated *eagerly* — the string is formatted even if the log level
means the message will be discarded. `%`-style logging passes the format string and
arguments separately; the logging framework only formats the string if the message
will actually be emitted. For expensive-to-format objects, this is a significant
performance difference.

```python
import logging

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.WARNING)

# BAD — expensive computation happens even if DEBUG is disabled:
logger.debug(f"Processing user: {user.full_profile_summary()}")  # eager eval!

# GOOD — format string evaluated only if DEBUG level is active:
logger.debug("Processing user: %s", user.full_profile_summary())
# But: user.full_profile_summary() is STILL called! To avoid even that:

if logger.isEnabledFor(logging.DEBUG):
    logger.debug("Processing user: %s", user.full_profile_summary())

# Another pattern — lazy string:
class LazyFormat:
    def __init__(self, func): self.func = func
    def __str__(self): return self.func()

logger.debug("Result: %s", LazyFormat(lambda: expensive_repr(data)))
```

---

## Trap 6: "Optional[T] is the same as Union[T, None] in older Python"

**What most people say:**
"`Optional[int]` is a special type that means the value might not be there."

**Correct answer:**
`Optional[T]` is literally defined as `Union[T, None]`. They are completely equivalent
at runtime and to type checkers. Since Python 3.10, you can also write `T | None`
with the pipe operator. The `Optional` name is sometimes considered less clear because
it implies the *parameter* is optional, whereas it really means the *value* can be None.

```python
from typing import Optional, Union

# These three are identical to type checkers:
def f1(x: Optional[int]) -> Optional[str]: ...
def f2(x: Union[int, None]) -> Union[str, None]: ...
def f3(x: int | None) -> str | None: ...  # Python 3.10+

# Common misconception — Optional does NOT mean "has a default":
def process(data: Optional[list]) -> None:
    # data is required (no default), but can be None
    if data is None:
        data = []

# Optional with a default — that's a different concept:
def process(data: Optional[list] = None) -> None:
    # data has a DEFAULT of None AND can be None
    if data is None:
        data = []
```

---

## Trap 7: "pathlib.Path('/a') / '/b' == pathlib.Path('/b')"

**What most people say:**
"You can join paths with `/` operator and it works like `os.path.join`."

**Correct answer:**
This is actually TRUE but surprising: `Path('/a') / '/b'` gives `Path('/b')`, not
`Path('/a/b')`. When the right operand is an absolute path (starts with `/`), it
discards the left side — exactly like `os.path.join('/a', '/b')`. This can cause
silent bugs when joining user-provided paths.

```python
from pathlib import Path

Path('/home/user') / 'documents'       # Path('/home/user/documents')  OK
Path('/home/user') / '/etc/passwd'     # Path('/etc/passwd')  — left side DISCARDED!

# Same behaviour as os.path.join:
import os.path
os.path.join('/home/user', '/etc/passwd')  # '/etc/passwd'

# Safe joining when input might be absolute:
base = Path('/home/user')
user_input = '/etc/passwd'

# Option 1: strip leading slash
safe = base / user_input.lstrip('/')   # Path('/home/user/etc/passwd')

# Option 2: use relative_to check
user_path = Path(user_input)
if user_path.is_absolute():
    raise ValueError("Absolute paths not allowed")
safe = base / user_path
```

---

## Trap 8: "Counter subtraction removes zero and negative counts"

**What most people say:**
"`Counter` automatically removes items when their count drops to zero or below."

**Correct answer:**
Counter arithmetic uses two different subtraction operations with different semantics.
`-=` (in-place) and `-` (creating a new Counter) behave differently — `subtract()`
KEEPS zero and negative counts, while `-` DISCARDS them.

```python
from collections import Counter

c = Counter(a=3, b=1)
c.subtract(Counter(a=1, b=2, c=5))  # subtract() keeps negatives
print(dict(c))   # {'a': 2, 'b': -1, 'c': -5}

c = Counter(a=3, b=1)
c -= Counter(a=1, b=2, c=5)         # -= discards zeros and negatives
print(dict(c))   # {'a': 2}  ← b and c removed because ≤ 0

# The + operator also discards ≤ 0 (useful for "clean" arithmetic):
c1 = Counter(a=3, b=-1)
+c1   # Counter({'a': 3})  — discards negatives

# elements() only yields positive counts:
c = Counter(a=2, b=0, c=-1)
list(c.elements())  # ['a', 'a']  — only a (count=2)
```

---

## Trap 9: "enum.Enum members are the same as their values"

**What most people say:**
"An Enum member IS its value, so you can compare them directly."

**Correct answer:**
`enum.Enum` members are NOT their values — they are instances of the Enum class.
`Color.RED == "red"` is `False` even if `Color.RED.value == "red"`. Use `IntEnum`
if you need an enum that IS an int (for backwards compatibility). Using `is` or `==`
on Enum members compares the member objects, not their values.

```python
from enum import Enum, IntEnum

class Color(Enum):
    RED   = "red"
    GREEN = "green"

Color.RED == "red"        # False — member vs string
Color.RED.value == "red"  # True  — access the value
Color.RED is Color.RED    # True  — singleton guarantee
Color.RED == Color.RED    # True  — same singleton

# IntEnum — IS the int:
class Priority(IntEnum):
    LOW  = 1
    HIGH = 5

Priority.HIGH == 5         # True — IntEnum behaves like int
Priority.HIGH > Priority.LOW  # True — numeric comparison works
sorted([Priority.HIGH, Priority.LOW])  # [<Priority.LOW: 1>, <Priority.HIGH: 5>]

# Accessing by value (lookup):
Color("red")    # Color.RED
Color("purple") # ValueError: 'purple' is not a valid Color
```

---

## Trap 10: "TypedDict enforces types at runtime"

**What most people say:**
"`TypedDict` validates that the dict has the right keys and types at runtime."

**Correct answer:**
`TypedDict` is a PURELY STATIC type-checking construct. At runtime, a `TypedDict` is
just a plain `dict` — no validation, no enforcement. It is a hint to type checkers
(mypy, pyright, Pylance) only. If you need runtime validation, use `pydantic`,
`marshmallow`, or `@dataclass` with `__post_init__`.

```python
from typing import TypedDict

class UserConfig(TypedDict):
    name: str
    age: int
    admin: bool

# This passes type checking — everything is correct
config: UserConfig = {"name": "Alice", "age": 30, "admin": False}

# This ALSO works at runtime — TypedDict doesn't validate!
bad_config: UserConfig = {"name": 123, "age": "old", "extra_key": True}
# No error! TypedDict is just a dict at runtime.

type(config)   # <class 'dict'>  — not a special TypedDict class

# For runtime validation, use pydantic:
from pydantic import BaseModel

class UserConfigModel(BaseModel):
    name: str
    age: int
    admin: bool

UserConfigModel(name=123, age="old", admin=False)
# ValidationError — pydantic coerces/validates at runtime
```

---

## Trap 11: "logging.basicConfig() can be called multiple times"

**What most people say:**
"Call `basicConfig()` in each module that needs logging configuration."

**Correct answer:**
`logging.basicConfig()` has NO EFFECT if the root logger already has handlers
configured. It is a "first call wins" function — subsequent calls are silently
ignored. This means if a third-party library calls `basicConfig()` before your code
does, your configuration will be silently ignored. Use explicit handler and logger
configuration for reliable logging setup in libraries.

```python
import logging

logging.basicConfig(level=logging.DEBUG)  # first call — sets up root logger
logging.basicConfig(level=logging.WARNING)  # second call — IGNORED silently!
logging.getLogger().level  # still DEBUG, not WARNING

# Reliable configuration:
logger = logging.getLogger("myapp")
logger.setLevel(logging.DEBUG)
handler = logging.StreamHandler()
handler.setLevel(logging.DEBUG)
handler.setFormatter(logging.Formatter("%(asctime)s %(name)s %(levelname)s %(message)s"))
logger.addHandler(handler)
logger.propagate = False  # don't send to root logger (avoids duplicates)
```
