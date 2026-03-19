# Chapter 03 — Functional Programming: Diagram Explanations

---

## Diagram 1: Decorator Execution Flow

```
SOURCE CODE:
  @timed
  def fetch(url): ...

EQUIVALENT TO:
  def fetch(url): ...
  fetch = timed(fetch)    ← decorator called at DEFINITION time

CALL TIME:
  fetch("https://api.example.com")
         │
         ▼
  wrapper(*args, **kwargs)     ← outer function from decorator
         │
    ┌────┴────────────────────────────────────────────┐
    │  BEFORE code (logging, timing, auth check...)   │
    └────┬────────────────────────────────────────────┘
         │
         ▼
  original_fetch("https://api.example.com")   ← the real function
         │
         ▼ (returns result)
    ┌────┴────────────────────────────────────────────┐
    │  AFTER code (log result, record time, cleanup)  │
    └────┬────────────────────────────────────────────┘
         │
         ▼
  return result to original caller

Parametrised decorator — THREE levels:

  @retry(times=3)        ← Level 1 called at definition time
  def unstable(): ...

  Level 1: retry(times=3)      → returns Level 2
  Level 2: decorator(unstable) → returns Level 3 (wrapper)
  Level 3: wrapper(*a, **kw)   → called at invocation time
```

---

## Diagram 2: Generator Execution — Pause and Resume

```
def count_up(n):          Caller
    print("start")
    for i in range(n):    gen = count_up(3)   ← body NOT executed yet
        yield i
    print("done")

Execution timeline:
                                     Generator          Caller
  gen = count_up(3)    ────────────►  (created,         gen object
                                       suspended)       returned

  next(gen)            ────────────►  print("start")
                                      i = 0
                                      yield 0   ─────►  receives 0
                                      (suspended)

  next(gen)            ────────────►  resume after yield
                                      i = 1
                                      yield 1   ─────►  receives 1
                                      (suspended)

  next(gen)            ────────────►  resume
                                      i = 2
                                      yield 2   ─────►  receives 2
                                      (suspended)

  next(gen)            ────────────►  resume
                                      loop ends
                                      print("done")
                                      return (implicit)  raises StopIteration ──► caller catches it (for loop ends)

State saved across yields:
  ┌───────────────────────────────────────────────┐
  │  Local variables: n=3, i=<current>            │
  │  Instruction pointer: <line after yield>      │
  │  Stack frame: preserved                       │
  └───────────────────────────────────────────────┘
```

---

## Diagram 3: Iterator Protocol — `for` Loop Desugaring

```
for item in my_list:        IS THE SAME AS:
    process(item)
                            iterator = iter(my_list)    → calls my_list.__iter__()
                            while True:
                                try:
                                    item = next(iterator) → calls iterator.__next__()
                                    process(item)
                                except StopIteration:
                                    break

Object taxonomy:
  ┌─────────────────────────────────────────────────────────────────┐
  │  Iterable                                                        │
  │  ┌ has __iter__() ──────────────────────────────────────────┐   │
  │  │                                                           │   │
  │  │  list, tuple, str, dict, set, range, file objects, ...   │   │
  │  │                                                           │   │
  │  │  ┌─────────────────────────────────────────────────┐    │   │
  │  │  │  Iterator (also an iterable)                     │    │   │
  │  │  │  ┌ has __iter__() AND __next__() ─────────────┐ │    │   │
  │  │  │  │                                             │ │    │   │
  │  │  │  │  iter(list), iter(range), map(), filter()  │ │    │   │
  │  │  │  │  zip(), enumerate()                        │ │    │   │
  │  │  │  │                                             │ │    │   │
  │  │  │  │  ┌─────────────────────────────────────┐  │ │    │   │
  │  │  │  │  │  Generator Iterator                  │  │ │    │   │
  │  │  │  │  │  (also has .send() .throw() .close())│  │ │    │   │
  │  │  │  │  │  — generator functions               │  │ │    │   │
  │  │  │  │  │  — generator expressions             │  │ │    │   │
  │  │  │  │  └─────────────────────────────────────┘  │ │    │   │
  │  │  │  └────────────────────────────────────────────┘ │    │   │
  │  │  └─────────────────────────────────────────────────┘    │   │
  │  └───────────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────────┘

Key difference: Iterables can be re-iterated (iter() returns fresh iterator).
               Iterators are single-pass — once exhausted, they stay exhausted.
```

---

## Diagram 4: yield from — Transparent Delegation

```
Without yield from:                   With yield from:
  def outer():                          def outer():
    for item in inner():                  yield from inner()
        yield item

  outer() ──► inner()                   outer() ──► inner()
    next(outer) calls:                    next(outer) calls:
      inner.__next__()                      inner.__next__()  DIRECTLY
    but send/throw NOT forwarded           send/throw forwarded transparently

Communication channels:

  WITHOUT yield from:                   WITH yield from:
  Caller ──► outer ──► inner            Caller ──────────────► inner
             (broken chain:             (transparent:
              send/throw lost)           full duplex)
  Caller ◄── outer ◄── inner            Caller ◄────────────── inner
                                         outer captures
                                         return value of inner

yield from semantics:
  result = yield from subgen

  ├── Yields every value subgen yields to the caller
  ├── Sends every value caller sends to subgen
  ├── Throws every exception caller throws into subgen
  ├── When subgen raises StopIteration(value), result = value
  └── Returns result in outer generator
```

---

## Diagram 5: functools.lru_cache — LRU Cache Structure

```
@lru_cache(maxsize=4)
def f(x): ...

Call history: f(1), f(2), f(3), f(4), f(2), f(5)

After f(1), f(2), f(3), f(4):    [MRU] 4 ← 3 ← 2 ← 1 [LRU]
                                   Cache full (maxsize=4)

After f(2) (cache hit, 2 promoted): [MRU] 2 ← 4 ← 3 ← 1 [LRU]

After f(5) (cache miss, 1 evicted): [MRU] 5 ← 2 ← 4 ← 3 [LRU]
                                     1 was LRU → evicted

Cache storage: dict mapping (args, kwargs) tuple → cached return value
               Doubly linked list for LRU ordering

CacheInfo attributes:
  hits     — number of cache hits (returned from cache)
  misses   — number of cache misses (function actually called)
  maxsize  — configured maximum size (None = unlimited)
  currsize — current number of cached entries

When to use lru_cache:
  ✓ Pure function (same inputs → always same output)
  ✓ Expensive computation (fibonacci, parsing, DB lookup)
  ✓ Hashable arguments (int, str, tuple, frozenset)
  ✗ Side effects (file writes, network calls with state)
  ✗ Time/random dependent results
  ✗ Mutable arguments (list, dict → TypeError)
```

---

## Diagram 6: Higher-Order Functions — Composition Patterns

```
Higher-order functions: functions that take or return functions

┌─────────────────────────────────────────────────────────────────────┐
│  PATTERN           EXAMPLE                   DESCRIPTION             │
├─────────────────────────────────────────────────────────────────────┤
│  map               map(func, iterable)        apply func to each     │
│  filter            filter(pred, iterable)     keep items where True  │
│  reduce            reduce(func, iterable)     fold/accumulate        │
│  sorted            sorted(items, key=func)    sort with key func     │
│  partial           partial(func, arg)         pre-fill arguments     │
│  lru_cache         lru_cache(func)            memoize results        │
│  decorator         @decorator def f           wrap behaviour         │
└─────────────────────────────────────────────────────────────────────┘

Function composition pipeline:
  raw_data
     │
     ▼
  map(parse)          ← transform each element
     │
     ▼
  filter(is_valid)    ← remove invalid elements
     │
     ▼
  map(transform)      ← transform again
     │
     ▼
  sorted(key=score)   ← order results
     │
     ▼
  list(...)           ← materialise (optional)

All steps are lazy until materialised → memory efficient for large datasets

compose() implementation using reduce:
  from functools import reduce

  def compose(*funcs):
      """compose(f, g, h)(x) == f(g(h(x)))"""
      return reduce(lambda f, g: lambda *a, **kw: f(g(*a, **kw)), funcs)

  pipeline = compose(str.upper, str.strip, str.lower)
  pipeline("  Hello World  ")   # "HELLO WORLD"
```
