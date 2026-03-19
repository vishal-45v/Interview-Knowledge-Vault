# Chapter 05 — Standard Library: Structured Answers

---

## Q1: collections module — key types and when to use each

**Core answer:**

```python
from collections import defaultdict, Counter, OrderedDict, deque, namedtuple, ChainMap

# --- defaultdict ---
# Automatically creates missing keys using a factory function

word_count = defaultdict(int)
for word in "the quick brown fox jumps over the lazy dog".split():
    word_count[word] += 1   # no KeyError for first occurrence

# Nested defaultdict:
graph = defaultdict(list)
graph["A"].append("B")
graph["A"].append("C")
# {'A': ['B', 'C']}  — clean adjacency list

# --- Counter ---
# Specialised dict for counting hashable objects

text = "mississippi"
c = Counter(text)
# Counter({'i': 4, 's': 4, 'p': 2, 'm': 1})

c.most_common(3)    # [('i', 4), ('s', 4), ('p', 2)]
c["z"]              # 0 — missing elements return 0 (not KeyError)

# Arithmetic:
a = Counter(a=3, b=1)
b = Counter(a=1, b=2, c=5)
a + b   # Counter({'c': 5, 'a': 4, 'b': 3})  — combine
a - b   # Counter({'a': 2})  — subtract, drop ≤ 0
a & b   # Counter({'a': 1, 'b': 1})  — intersection (min)
a | b   # Counter({'c': 5, 'a': 3, 'b': 2})  — union (max)

# --- deque ---
# Double-ended queue — O(1) appends and pops from BOTH ends

dq = deque([1, 2, 3], maxlen=5)
dq.appendleft(0)    # [0, 1, 2, 3]
dq.append(4)        # [0, 1, 2, 3, 4]
dq.append(5)        # [1, 2, 3, 4, 5] — 0 evicted (maxlen=5)
dq.popleft()        # 1; deque: [2, 3, 4, 5]
dq.rotate(2)        # rotate right by 2: [4, 5, 2, 3]

# Use deque instead of list when:
# - You need fast operations at BOTH ends
# - list.pop(0) is O(n); deque.popleft() is O(1)

# --- namedtuple ---
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
p.x         # 3  — named access
p[0]        # 3  — index access (it IS a tuple)
p._asdict() # {'x': 3, 'y': 4}
p._replace(x=10)  # Point(x=10, y=4)  — immutable; returns new
```

---

## Q2: contextlib — contextmanager, suppress, ExitStack

**Core answer:**

```python
from contextlib import contextmanager, suppress, ExitStack

# --- @contextmanager ---
@contextmanager
def managed_connection(host: str, port: int):
    """Generator-based context manager — simpler than class."""
    conn = connect(host, port)
    try:
        yield conn          # __enter__ return value
    except ConnectionError:
        conn.reset()
        raise               # re-raise after cleanup
    finally:
        conn.close()        # always runs

with managed_connection("db.example.com", 5432) as conn:
    results = conn.query("SELECT * FROM users")

# --- suppress ---
# Cleaner than try/except/pass for expected exceptions

# Instead of:
try:
    os.remove("file.txt")
except FileNotFoundError:
    pass

# Use:
with suppress(FileNotFoundError):
    os.remove("file.txt")

# Multiple exception types:
with suppress(FileNotFoundError, PermissionError):
    os.remove("protected_file.txt")

# --- ExitStack ---
# Dynamically compose multiple context managers

def process_files(filenames):
    with ExitStack() as stack:
        # Open an unknown number of files — all will be closed on exit
        files = [stack.enter_context(open(f)) for f in filenames]
        for file in files:
            process(file)
    # All files closed here, even if an exception occurred

# Conditional context manager:
with ExitStack() as stack:
    if verbose:
        stack.enter_context(timer_context())
    if debug:
        stack.enter_context(debug_context())
    do_work()
```

---

## Q3: pathlib — modern filesystem operations

**Core answer:**

```python
from pathlib import Path

# Construction:
p = Path("/home/user/documents/report.pdf")
p = Path.home() / "documents" / "report.pdf"  # OS-independent

# Navigation:
p.parent       # Path('/home/user/documents')
p.name         # 'report.pdf'
p.stem         # 'report'
p.suffix       # '.pdf'
p.suffixes     # ['.pdf']
p.parts        # ('/', 'home', 'user', 'documents', 'report.pdf')

# Checks:
p.exists()     # True/False
p.is_file()    # True if exists and is a file
p.is_dir()     # True if exists and is a directory

# Reading/writing:
content = p.read_text(encoding="utf-8")   # one-shot text read
p.write_text("new content", encoding="utf-8")
data = p.read_bytes()   # binary
p.write_bytes(data)

# Creating:
Path("new_dir").mkdir(parents=True, exist_ok=True)

# Listing:
src = Path("src")
list(src.glob("*.py"))              # non-recursive
list(src.rglob("*.py"))             # recursive — all .py files
list(src.glob("**/*.py"))           # same as rglob

# Path arithmetic:
base = Path("/app")
config = base / "config" / "settings.json"  # /app/config/settings.json

# Conversion:
str(p)              # '/home/user/documents/report.pdf'
p.resolve()         # absolute, normalised (resolves symlinks)

# vs os.path (before pathlib):
import os.path
old_path = os.path.join("/home/user/documents", "report.pdf")
old_parent = os.path.dirname(old_path)
old_exists = os.path.exists(old_path)
# pathlib is object-oriented; os.path is string-function-based
```

---

## Q4: typing module — type hints for modern Python

**Core answer:**

```python
from typing import (
    Optional, Union, List, Dict, Tuple, Set, Any,
    TypeVar, Generic, Protocol, TypedDict, Literal,
    Callable, Iterator, Generator, overload
)
from typing import TYPE_CHECKING

# Basic generics (Python 3.9+: use built-in list[T], dict[K,V]):
def get_names(users: List[Dict[str, Any]]) -> List[str]:
    return [u["name"] for u in users]

# Optional and Union:
def find_user(uid: int) -> Optional[str]:  # may return None
    ...

def parse_value(s: str) -> Union[int, float, None]:  # one of three types
    ...

# Python 3.10+ pipe syntax:
def find_user_new(uid: int) -> str | None: ...

# TypeVar — generic functions:
T = TypeVar("T")

def first(items: List[T]) -> Optional[T]:
    return items[0] if items else None

first([1, 2, 3])       # inferred return type: Optional[int]
first(["a", "b"])      # inferred return type: Optional[str]

# Protocol — structural typing (duck typing + type checking):
class Readable(Protocol):
    def read(self, n: int = -1) -> str: ...

def process_stream(stream: Readable) -> str:
    return stream.read()

# Works with any object that has .read() — no inheritance needed:
import io
process_stream(io.StringIO("hello"))  # OK
process_stream(open("file.txt"))      # OK
# process_stream(42)                  # mypy error: int has no .read()

# TypedDict:
class Config(TypedDict):
    host: str
    port: int
    debug: bool

class OptionalConfig(TypedDict, total=False):  # all keys optional
    timeout: int
    max_retries: int

# Literal — exact value types:
Mode = Literal["read", "write", "append"]

def open_file(path: str, mode: Mode) -> None: ...
open_file("file.txt", "read")    # OK
open_file("file.txt", "delete")  # mypy error: not a valid Mode

# Callable:
Handler = Callable[[int, str], bool]  # takes int, str; returns bool

def register_handler(h: Handler) -> None: ...
```

---

## Q5: enum — type-safe constants

**Core answer:**

```python
from enum import Enum, IntEnum, Flag, auto

# Basic Enum:
class Status(Enum):
    PENDING  = "pending"
    ACTIVE   = "active"
    INACTIVE = "inactive"
    DELETED  = "deleted"

# Usage:
Status.PENDING           # <Status.PENDING: 'pending'>
Status.PENDING.value     # 'pending'
Status.PENDING.name      # 'PENDING'

# Lookup by value:
Status("active")         # Status.ACTIVE

# Comparison — identity not value:
Status.PENDING == "pending"   # False
Status.PENDING == Status.PENDING  # True
Status.PENDING is Status.PENDING  # True — enum members are singletons

# Iteration:
for s in Status:
    print(s.name, s.value)

# auto() — auto-assign values:
class Direction(Enum):
    NORTH = auto()   # 1
    SOUTH = auto()   # 2
    EAST  = auto()   # 3
    WEST  = auto()   # 4

# IntEnum — behaves as int (backwards compatibility):
class Priority(IntEnum):
    LOW    = 1
    MEDIUM = 5
    HIGH   = 10

Priority.HIGH > Priority.LOW  # True — integer comparison
sorted_tasks = sorted(tasks, key=lambda t: t.priority)  # works

# Flag — combinable bit flags:
class Permission(Flag):
    READ    = auto()  # 1
    WRITE   = auto()  # 2
    EXECUTE = auto()  # 4

user_perms = Permission.READ | Permission.WRITE
Permission.READ in user_perms    # True
Permission.EXECUTE in user_perms # False

# Enum with methods:
class Color(Enum):
    RED   = (255, 0, 0)
    GREEN = (0, 255, 0)
    BLUE  = (0, 0, 255)

    def __init__(self, r, g, b):
        self.r = r; self.g = g; self.b = b

    @property
    def hex(self) -> str:
        return f"#{self.r:02x}{self.g:02x}{self.b:02x}"

Color.RED.hex    # '#ff0000'
```
