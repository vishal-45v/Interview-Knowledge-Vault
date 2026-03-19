# Chapter 05 — Standard Library: Diagram Explanations

---

## Diagram 1: collections Module — Type Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│  TYPE           BASE      MUTABLE  ORDERED  HASHABLE  KEY FEATURE        │
├─────────────────────────────────────────────────────────────────────────┤
│  dict           dict      yes      3.7+ yes  no       standard K/V       │
│  defaultdict    dict      yes      3.7+ yes  no       auto-create keys   │
│  OrderedDict    dict      yes      yes       no       move_to_end, eq    │
│  Counter        dict      yes      3.7+ yes  no       count arithmetic   │
│  ChainMap       MutableMapping yes  yes  no  layered lookup              │
├─────────────────────────────────────────────────────────────────────────┤
│  list           list      yes      yes       no       dynamic array      │
│  deque          deque     yes      yes       no       O(1) both ends     │
├─────────────────────────────────────────────────────────────────────────┤
│  tuple          tuple     no       yes       yes*     immutable seq      │
│  namedtuple     tuple     no       yes       yes*     named field access │
└─────────────────────────────────────────────────────────────────────────┘
* hashable only if all contained elements are hashable

deque vs list performance:
  Operation         list        deque
  append(x)         O(1) amort  O(1)
  appendleft(x)     O(n)        O(1)
  pop()             O(1) amort  O(1)
  popleft()         O(n)        O(1)
  insert(i, x)      O(n)        O(n)
  index access [i]  O(1)        O(n)
```

---

## Diagram 2: logging Hierarchy — Logger Tree

```
                        root logger (logging.getLogger())
                               │
                    propagate=True by default
                               │
             ┌─────────────────┼──────────────────┐
             │                 │                  │
        "myapp"            "requests"          "sqlalchemy"
        logger             logger              logger
             │
      ┌──────┴──────┐
      │             │
  "myapp.web"   "myapp.db"
   logger        logger

Message flow (if propagate=True):
  "myapp.db" logger emits WARNING
      │
      ▼ (if no handler at myapp.db level, or propagate=True)
  "myapp" logger receives it
      │
      ▼ (propagate=True)
  root logger receives it
      │
      ▼
  root's handlers emit it (console/file)

Handler levels act as additional filters:
  Logger level ──► Handler level ──► Formatter ──► Output

  Logger level=DEBUG means: pass DEBUG and above to handlers
  Handler level=WARNING means: of those, only emit WARNING and above

Log levels (numeric values):
  NOTSET    =  0
  DEBUG     = 10  (detailed diagnostic info)
  INFO      = 20  (routine events)
  WARNING   = 30  (something unexpected; default level)
  ERROR     = 40  (a function failed)
  CRITICAL  = 50  (program may not continue)
```

---

## Diagram 3: pathlib.Path — Object-Oriented Filesystem API

```
Path object anatomy:
  /home/user/projects/myapp/src/main.py

  ├── drive     → '' (on Unix) or 'C:' (on Windows)
  ├── root      → '/'
  ├── anchor    → '/'
  ├── parts     → ('/', 'home', 'user', 'projects', 'myapp', 'src', 'main.py')
  ├── parent    → Path('/home/user/projects/myapp/src')
  ├── parents   → [src, myapp, projects, user, home, /]
  ├── name      → 'main.py'
  ├── stem      → 'main'
  └── suffix    → '.py'

Path navigation:
  p = Path('/home/user')
  p / 'docs' / 'report.pdf'
      → Path('/home/user/docs/report.pdf')

  p.parent    → Path('/home')
  p.parent.parent → Path('/')

Common operations:
  ┌────────────────────────────────────────────────────────────────┐
  │  os.path (old)                   pathlib (new)                  │
  ├────────────────────────────────────────────────────────────────┤
  │  os.path.join(a, b)          →   Path(a) / b                  │
  │  os.path.exists(p)           →   Path(p).exists()             │
  │  os.path.isfile(p)           →   Path(p).is_file()            │
  │  os.path.isdir(p)            →   Path(p).is_dir()             │
  │  os.path.basename(p)         →   Path(p).name                 │
  │  os.path.dirname(p)          →   Path(p).parent               │
  │  os.path.splitext(p)         →   (p.stem, p.suffix)           │
  │  open(p, 'r').read()         →   Path(p).read_text()          │
  │  os.makedirs(p, exist_ok=T)  →   Path(p).mkdir(parents=T,     │
  │                                              exist_ok=T)       │
  │  glob.glob('**/*.py', r=T)   →   Path(p).rglob('*.py')       │
  └────────────────────────────────────────────────────────────────┘
```

---

## Diagram 4: typing Module — Type Hierarchy for Common Annotations

```
typing constructs:

  Concrete types:
  ┌─────────────────────────────────────────────────────────────────┐
  │  int, str, float, bool, bytes  — built-in types                 │
  │  List[int]   → list of ints                                     │
  │  Dict[str, int] → dict with str keys, int values               │
  │  Tuple[int, str, float] → fixed-length tuple                   │
  │  Tuple[int, ...] → variable-length tuple of ints               │
  │  Set[str]    → set of strings                                   │
  │  FrozenSet[str] → frozenset of strings                         │
  └─────────────────────────────────────────────────────────────────┘

  Combining types:
  ┌─────────────────────────────────────────────────────────────────┐
  │  Optional[T]     = Union[T, None]  — value or None             │
  │  Union[A, B, C]  — one of several types                        │
  │  A | B | C       — Python 3.10+ shorthand for Union            │
  └─────────────────────────────────────────────────────────────────┘

  Callables:
  ┌─────────────────────────────────────────────────────────────────┐
  │  Callable[[ArgType1, ArgType2], ReturnType]                     │
  │  Callable[..., ReturnType] — any arguments                     │
  └─────────────────────────────────────────────────────────────────┘

  Generics with TypeVar:
  ┌─────────────────────────────────────────────────────────────────┐
  │  T = TypeVar('T')                                               │
  │  def identity(x: T) -> T: return x                             │
  │     — return type matches input type exactly                    │
  │                                                                 │
  │  T = TypeVar('T', int, float)  — constrained TypeVar           │
  │     — T can only be int or float                                │
  │                                                                 │
  │  T = TypeVar('T', bound=Comparable)  — bounded TypeVar         │
  │     — T must be a subtype of Comparable                         │
  └─────────────────────────────────────────────────────────────────┘

  Structural typing:
  ┌─────────────────────────────────────────────────────────────────┐
  │  Protocol  — define required interface structurally             │
  │  TypedDict — typed dict with known keys                         │
  │  Literal   — exact string/int values as types                  │
  │  Final     — constant that cannot be reassigned                 │
  └─────────────────────────────────────────────────────────────────┘

  ABC-based vs Protocol-based:
    ABC (nominal):      isinstance(obj, MyABC)     → True if inherits
    Protocol (structural): obj satisfies protocol → True if has methods
                           (checked by type checker, not isinstance by default)
```

---

## Diagram 5: JSON Serialisation — What Converts and What Doesn't

```
Python type         JSON type         Notes
─────────────────────────────────────────────────────────────────────
dict               object {}          keys must be strings (or auto-str)
list, tuple        array  []          tuple becomes array (loses type)
str                string ""
int                number
float              number             Infinity, NaN → not valid JSON!
True               true
False              false
None               null
─────────────────────────────────────────────────────────────────────
datetime           ❌ TypeError       must implement custom encoder
Decimal            ❌ TypeError       must implement custom encoder
set, frozenset     ❌ TypeError       convert to list first
bytes              ❌ TypeError       encode to base64 or decode to str
custom objects     ❌ TypeError       implement __dict__ or custom encoder
─────────────────────────────────────────────────────────────────────

Custom encoder pattern:
  class CustomEncoder(json.JSONEncoder):
      def default(self, obj):
          if isinstance(obj, datetime): return obj.isoformat()
          if isinstance(obj, Decimal):  return float(obj)
          if isinstance(obj, set):      return sorted(obj)
          return super().default(obj)  # raise TypeError for unknowns

  json.dumps(data, cls=CustomEncoder)

Round-trip safety:
  Python → JSON → Python

  dict   → object → dict    ✓ (keys may become str if they were int)
  tuple  → array  → list    ✗ (tuple becomes list after round-trip)
  int    → number → int     ✓
  float  → number → float   ✓ (precision may change)
  None   → null   → None    ✓
```

---

## Diagram 6: contextlib — Context Manager Patterns

```
Three ways to write a context manager:

1. Class-based (explicit __enter__ / __exit__):
   ┌──────────────────────────────────────────────────────────────┐
   │  class CM:                                                    │
   │      def __enter__(self):                                     │
   │          setup()                                             │
   │          return resource                                     │
   │                                                              │
   │      def __exit__(self, exc_type, exc_val, exc_tb):          │
   │          teardown()                                          │
   │          return False  # False: don't suppress exceptions    │
   └──────────────────────────────────────────────────────────────┘

2. Generator-based (@contextmanager):
   ┌──────────────────────────────────────────────────────────────┐
   │  @contextmanager                                             │
   │  def cm():                                                   │
   │      setup()                        # ← __enter__ body      │
   │      try:                                                    │
   │          yield resource             # ← suspension point    │
   │      finally:                                                │
   │          teardown()                 # ← __exit__ body       │
   └──────────────────────────────────────────────────────────────┘

3. ExitStack (dynamic composition):
   ┌──────────────────────────────────────────────────────────────┐
   │  with ExitStack() as stack:                                  │
   │      resources = [                                           │
   │          stack.enter_context(open(f))                        │
   │          for f in dynamic_file_list                          │
   │      ]                                                       │
   │      stack.callback(cleanup_fn)    # arbitrary cleanup       │
   └──────────────────────────────────────────────────────────────┘

__exit__ arguments:
  exc_type  — type of exception (None if clean exit)
  exc_val   — exception instance
  exc_tb    — traceback object

  Return value:
    False / None  → re-raise exception (don't suppress)
    True          → suppress exception (swallow it)

  contextlib.suppress internally returns True for specified exceptions:

  class suppress:
      def __exit__(self, exc_type, ...):
          return exc_type is not None and issubclass(exc_type, self.exceptions)
```
